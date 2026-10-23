

## Surfaceflinger就绪

SurfaceFlinger进程在系统启动过程中，当init进程执行到相应的启动阶段后`/system/bin/surfaceflinger`被执行，入口在`main_surfaceflinger.cpp`，所有线程都从[`main()`](./SurfaceFlinger线程.md#SFinit)开始派生

