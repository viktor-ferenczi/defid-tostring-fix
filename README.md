# MyDefinitionId.ToString performance and deadlock fix

This is a fix for the following Space Engineers tickets:
- [Deadlock](https://support.keenswh.com/spaceengineers/pc/topic/27997-servers-deadlocked-on-load)
- [Slowness](https://support.keenswh.com/spaceengineers/pc/topic/24210-performance-pre-calculate-or-cache-mydefinitionid-tostring-results)

Plugins:
- [Torch](https://torchapi.com/plugins/view/e368127c-0676-4ff0-9e8c-f32207dcb12a)
- [Dedicated Server](https://github.com/viktor-ferenczi/defid-tostring-fix/releases)

## Background

Had to make this fix as a separate plugin, because that the `MyDefinitionId.ToString` overridden 
virtual method just could not be patched with Torch's patcher, 
so must have used Harmony for this fix.

This plugin completely replaces Keen's `MyDefinitionId.ToString` implementation, including its
cache which can just deadlock during the heavy load of world loading.

Tested the cache hit rate with a big world (Prometheus) saved in early 2020 when I worked on the 
[Performance Improvements](https://github.com/viktor-ferenczi/performance-improvements) plugin.

Debug log output (the release build does not log anything):
```log
02:49:32.4805 [INFO]   Keen: Game ready...
02:50:18.4774 [INFO]   DefIdToStringFix: L1: HitRate = 0.000% = 0/125020; ItemCount = 0 | L2: HitRate = 96.871% = 121108/125020; ItemCount = 3912
02:50:49.1695 [INFO]   DefIdToStringFix: L1: HitRate = 100.000% = 10727/10727; ItemCount = 3912 | L2: HitRate = 100.000% = 0/0; ItemCount = 3912
02:51:20.4694 [INFO]   DefIdToStringFix: L1: HitRate = 100.000% = 16344/16344; ItemCount = 3912 | L2: HitRate = 100.000% = 0/0; ItemCount = 3912
02:51:51.8590 [INFO]   DefIdToStringFix: L1: HitRate = 100.000% = 15818/15818; ItemCount = 3912 | L2: HitRate = 100.000% = 0/0; ItemCount = 3912
02:52:23.3482 [INFO]   DefIdToStringFix: L1: HitRate = 100.000% = 14292/14292; ItemCount = 3912 | L2: HitRate = 100.000% = 0/0; ItemCount = 3912
```

The cache hit rates are perfect.

## Theory of operation

The L1 cache has no lock at all, that's a read-only dictionary. 

The L2 cache collects any new strings not found in the L1 cache.

Every 27 seconds if there are any new entries in the L2 cache, then the whole L1 cache 
is re-generated from the L2 cache contents and the L1 cache is replaced by a single
atomic assignment. The old L1 cache is then garbage collected.

This update should only happen shortly after world loading and usually only at most once,
therefore it does not affect the server's normal operation (when players are joined) at all.

Both the L1 and L2 caches are cleared if `DropToStringCache` is called, that's Keen's
cache invalidation request on unloading worlds. Each world may have different set of
definition strings depending on the mods loaded.

The string formatting is equivalent to Keen's original implementation. No change there,
other than the code readability improvements applied to the decompiled code.