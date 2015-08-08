---
layout: post
title: DraweeView分析
category : fresco
tagline: "Supporting tagline"
tags : [fresco, android]
---
{% include JB/setup %}
#DraweeView分析
这是```DraweeView```的解释.使用这个view必须设置```DraweeHierarchy```,创建```DraweeHierarchy```是一件耗时又耗资源的操作,所以每个```DraweeView```只就创建一次```DraweeHierarchy```,为了显示一个图片
必须调用```setController```设置```DraweeController```,这个虽然是```ImageView```的子类但是不支持setImageXxx和setScaleType方法,后面可能会直接继承自View所以避免使用ImageView的方法和属性

    private void init(Context context) {
        mDraweeHolder = DraweeHolder.create(null, context);
    }

这是这个类的的初始化方法,这里初始化了一个```DraweeHolder```,DraweeHolder包含了```DraweeController```和```DraweeHierarchy```的实例,这个类的设计是为了```DraweeView```的解耦.换成别外一个类也可以使用.
每次为````DraweeView````更新图像时,会通过```DraweeView```的```setController()```的```super.setImageDrawable(mDraweeHolder.getTopLevelDrawable())```来设置图像

    public void setController(@Nullable DraweeController draweeController) {
      boolean wasAttached = mIsControllerAttached;
      if (wasAttached) {
        detachController();
      }

      // Clear the old controller
      if (mController != null) {
        mEventTracker.recordEvent(Event.ON_CLEAR_OLD_CONTROLLER);
        mController.setHierarchy(null);
      }
      mController = draweeController;
      if (mController != null) {
        mEventTracker.recordEvent(Event.ON_SET_CONTROLLER);
        mController.setHierarchy(mHierarchy);
      } else {
        mEventTracker.recordEvent(Event.ON_CLEAR_CONTROLLER);
      }

      if (wasAttached) {
        attachController();
      }
    }

调用这个方法会调用```attachController()```方法

    private void attachController() {
       if (mIsControllerAttached) {
         return;
       }
       mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
       mIsControllerAttached = true;
       if (mController != null &&
           mController.getHierarchy() != null) {
         mController.onAttach();
       }
     }
 ```attachController()```会调用```mController.onAttach()```调用这个方法会调用```submitRequest()```来发送一个请求,这里就开始发送http请求获取图片了.

    protected void submitRequest() {
    mEventTracker.recordEvent(Event.ON_DATASOURCE_SUBMIT);
    getControllerListener().onSubmit(mId, mCallerContext);
    mSettableDraweeHierarchy.setProgress(0, true);
    mIsRequestSubmitted = true;
    mHasFetchFailed = false;
    mDataSource = getDataSource();
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            // isFinished must be obtained before image, otherwise we might set intermediate result
            // as final image.
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(id, dataSource, image, progress, isFinished, wasImmediate);
            } else if (isFinished) {
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }
          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }
          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
        mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
    }

这里有一个DataSubscriber可以看作是一个观察者,这个后面在说
这个方法会调用```getDataSource()```,然后调用```PipelineDraeeeController```这个子类里的实现方法,现在来看看```PipelineDraeeeController```这个类
先来看一下构造方法

    public PipelineDraweeController(
        Resources resources,
        DeferredReleaser deferredReleaser,
        AnimatedDrawableFactory animatedDrawableFactory,
        Executor uiThreadExecutor,
        Supplier<DataSource<CloseableReference<CloseableImage>>> dataSourceSupplier,
        String id,
        Object callerContext) {
        super(deferredReleaser, uiThreadExecutor, id, callerContext);
      mResources = resources;
      mAnimatedDrawableFactory = animatedDrawableFactory;
      init(dataSourceSupplier);
    }

初始化这个类是```PipelineDraweeControllerFactory```的```newController()```new出来的,```newController```调用者是```PipelineDraweeControllerBuilder```的```obtainController()```,然后在调用的是```AbstractDraweeControllerBuilder```的```buildController()```,然后```build()```是最后的初始化
![draweecontroller初始化](http://wlanjie.github.io/blog/image/draweeController_init.jpg)
