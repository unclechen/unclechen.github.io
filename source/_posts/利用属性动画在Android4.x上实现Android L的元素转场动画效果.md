---
layout: post
title: 利用属性动画在Android4.x上实现Android L的元素转场动画效果（shared elements transition）
date: '2016-11-08'
tags:
  - Android
  - 动画
categories: 
  - 技术
---

# 一、背景

随着谷歌推出的MaterialDesign不断被各种实践，最近我也碰到这么一个需求，就是要求实现一个图片的转场效果。在第一个界面上，图片被点击后，会渐渐地滑动到第二个界面中去。

其实仔细观察一下Google相册也有用到这种效果，大概的效果图是这样的：

![transition-ui-demo](http://ww2.sinaimg.cn/large/801b780agw1f9kunqk6sqg20b40jre86.gif)

按照我的理解，这种效果要是直接把View拿出来复用就可以。但是经过学习（国内外各种大神的博客）和实践发现，里面有不少可以思考的地方。

<!-- more -->

根据官方介绍的[Transitions](https://developer.android.com/training/material/animations.html#Transitions)，在Android 5.0以后，可以使用**shared elements transition**来实现这个效果，非常方便。但是我们的App一般还需要向下兼容到4.x，所以在4.x上得想其它的办法实现。

下面我以上图的ImageView转场动画为例子，介绍一下是怎么实现这个效果的。（代码都在[这里](https://github.com/unclechen/ActivityTransitionDemo)）

> 注意：为了说明转场效果实现的核心内容，一些无关的东西都用了最简单的实现。

# 二、实现思路

## 1.入场

- （1）保存第一个Activity中ImageView（我们叫它originImageView）的位置信息、宽、高，然后把这些信息传给第二个Activity。
- （2）去掉Activity默认的转场动画。
- （3）进入第二个Activity之后，拿到第一个Activity传过来的ImageView的位置、宽、高信息，并在第二个Activity动态添加一个一模一样的ImageView（我叫它sourceImageView）。
- （4）在第二个Activity中，找到最终的ImageView（我叫它targetImageView），并取出它最终所在的位置。
- （5）对比sourceImageView和targetImageView的位置、大小等等各种**属性**的区别，然后使用属性动画将sourceImageView变换成targetImageView。
- （6）当动画结束时，显示出targetImageView，隐藏sourceImageView。

> 注意：这里当动画结束时，我们需要将sourceImageView的LayoutParams改成和targetImageView的LayoutParams一模一样，用于退出时做转场动画使用。
其实退场效果和入场效果是完全相反的步骤。

## 2.退场

- （1）将之前隐藏的sourceImageView显示出来，隐藏targetImageView。
- （2）通过属性动画将sourceImageView从当前的位置和宽、高大小，变换到刚进入第二个Activity时的状态。（这里的动画代码几乎一样，只是把开始值和结束值调换了位置）
- （3）动画结束时，关闭第二个Activity，去掉Activity的转场动画。

上面就是实现思路，其实很好理解。实现这个思路的重点，就在于属性动画的应用了。也就是上面提到的**入场的第5步**和**退场的第2步**，这里面用到的属性动画代码见下一章。

# 三、实现代码

## 1.入场

### （1）先复原出sourceImageView

```
    // 创建一个和第一个界面一模一样的ImageView，作为这个界面的sourceImageView
    private void initSourceImageView() {
        // 先动态创建出这个sourceImageView，把它添加到第二个界面的ContentView中。
        FrameLayout contentView = (FrameLayout) getWindow().getDecorView().findViewById(android.R.id.content);
        mSourceImageView = new ImageView(this);
        contentView.addView(mSourceImageView);

        // 读取第一个界面传过来的信息
        Bundle bundle = getIntent().getExtras();
        mRect = (Rect) getIntent().getParcelableExtra(IMAGE_ORIGIN_RECT);
        ImageView.ScaleType scaleType = (ImageView.ScaleType) bundle.getSerializable(IMAGE_SCALE_TYPE);
        mResId = bundle.getInt(IMAGE_RES_ID);

        // 设置为和第一个界面一样的图片
        mSourceImageView.setImageResource(mResId);
        mTargetImageView.setImageResource(mResId);

        // 设为和原来一样的裁剪模式
        mSourceImageView.setScaleType(scaleType);

        // 设置为和原来一样的位置
        FrameLayout.LayoutParams layoutParams = (FrameLayout.LayoutParams) mSourceImageView.getLayoutParams();
        layoutParams.width = mRect.width();
        layoutParams.height = mRect.height();
        layoutParams.setMargins(mRect.left, mRect.top, 0, 0);
    }

```

### （2）找到targetImageView的位置和宽高

```
    private void initImageEnterAnimation() {
        mTargetImageView.getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {

            @Override
            public boolean onPreDraw() {
                // 第一帧被绘制时，TargetImageView已经具有了实际的尺寸和位置，这是就应该开始播放动画。
                final int[] finalLocationOnTheScreen = new int[2];
                mTargetImageView.getLocationOnScreen(finalLocationOnTheScreen);
                mTargetLeft = finalLocationOnTheScreen[0];
                mTargetTop = finalLocationOnTheScreen[1];
                mTargetWidth = mTargetImageView.getWidth();
                mTargetHeight = mTargetImageView.getHeight();
                playEnteringAnimation(mTargetLeft, mTargetTop, mTargetWidth, mTargetHeight);
                mTargetImageView.getViewTreeObserver().removeOnPreDrawListener(this);
                return true;
            }
        });
    }

```

### （3）播放入场动画

```
    // 属性动画走起，将sourceImageView变换到targetImageView
    private void playEnteringAnimation(final int left, final int top, final int width, final int height) {
        // 1.改变ImageView的位置、宽高
        PropertyValuesHolder propertyLeft = PropertyValuesHolder.ofInt("left", mSourceImageView.getLeft(), left);
        PropertyValuesHolder propertyTop = PropertyValuesHolder.ofInt("top", mSourceImageView.getTop(), top);
        PropertyValuesHolder propertyRight = PropertyValuesHolder.ofInt("right", mSourceImageView.getRight(), left + width);
        PropertyValuesHolder propertyBottom = PropertyValuesHolder.ofInt("bottom", mSourceImageView.getBottom(), top + height);

        ObjectAnimator positionAnimator = ObjectAnimator.ofPropertyValuesHolder(mSourceImageView,
                propertyLeft, propertyTop, propertyRight, propertyBottom);
        positionAnimator.addListener(new Animator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animation) {

            }

            @Override
            public void onAnimationEnd(Animator animation) {
                // 为了退出动画，需要把sourceImageView的LayoutParams改成targetImageView
                FrameLayout.LayoutParams layoutParams = (FrameLayout.LayoutParams) mSourceImageView.getLayoutParams();
                layoutParams.height = height;
                layoutParams.width = width;
                layoutParams.setMargins(left, top, 0, 0);
            }

            @Override
            public void onAnimationCancel(Animator animation) {
            }

            @Override
            public void onAnimationRepeat(Animator animation) {

            }
        });

        // 2.ImageView的矩阵动画
        Matrix initMatrix = getImageMatrix(mSourceImageView);
        initMatrix.getValues(mInitImageMatrixValues);
        final Matrix endMatrix = getImageMatrix(mTargetImageView);
        mSourceImageView.setScaleType(ImageView.ScaleType.MATRIX);
        // ofObject()用法：传入自定义Property和Evaluator的用法
        ObjectAnimator matrixAnimator = ObjectAnimator.ofObject(mSourceImageView, ANIMATED_TRANSFORM_PROPERTY, new MatrixEvaluator(), initMatrix, endMatrix);

        // 3.顺便加个渐变动画
        ObjectAnimator fadeInAnimator = ObjectAnimator.ofFloat(mContainer, "alpha", 0.0f, 1.0f);

        // 4.一起播放上面的动画
        mEnteringAnimation = new AnimatorSet();
        mEnteringAnimation.setDuration(IMAGE_TRANSLATION_DURATION);
        mEnteringAnimation.setInterpolator(DEFAULT_INTERPOLATOR);
        mEnteringAnimation.addListener(new Animator.AnimatorListener() {

            @Override
            public void onAnimationCancel(Animator animation) {
                mEnteringAnimation = null;
            }

            @Override
            public void onAnimationRepeat(Animator animation) {

            }

            @Override
            public void onAnimationStart(Animator animation) {

            }

            @Override
            public void onAnimationEnd(Animator animation) {
                if (mEnteringAnimation != null) {
                    mEnteringAnimation = null;
                    mTargetImageView.setVisibility(View.VISIBLE);
                    mSourceImageView.setVisibility(View.INVISIBLE);
                }
            }
        });

        mEnteringAnimation.playTogether(positionAnimator, matrixAnimator, fadeInAnimator);
        mEnteringAnimation.start();
    }

```

### （4）2个关键的入场动画的说明

在（3）中，用到下面两个关键的动画：

- ObjectAnimator - positionAnimator：改变sourceImageView的top、left、right、bottom属性，动画的起始值就是sourceImageView的这4个属性，结束值就是targetImageView的这4个属性。

- ObjectAnimator - matrixAnimator：通过改变sourceImageView的Matrix，来改变其中显示的图片（drawable）的Bounds，从而使ImageView呈现出渐变效果。

介绍这两个关键动画的实现细节之前，需要具备属性动画的基础知识。如果不熟悉的话，建议先看下这几篇博客，里面详细地介绍了属性动画的各种用法。

- [ValueAnimator基本使用](http://blog.csdn.net/harvic880925/article/details/50525521)

- [ValueAnimator高级进阶（一）](http://blog.csdn.net/harvic880925/article/details/50546884)

- [ValueAnimator高级进阶（二）](http://blog.csdn.net/harvic880925/article/details/50549385)

- [ObjectAnimator基本使用](http://blog.csdn.net/harvic880925/article/details/50598322)

- [PropertyValuesHolder与Keyframe](http://blog.csdn.net/harvic880925/article/details/50752838)

- [联合动画的代码实现](http://blog.csdn.net/harvic880925/article/details/50759059)

---

下面介绍这两个关键的动画实现
 
<h4 style="color:#ff0000">关键动画之 ObjectAnimator - positionAnimator 实现：改变ImageView的位置和大小</h4>

我们知道，设置一个动画，就要给它设置起始值和结束值。所以我们的positionAnimator就需要设置sourceImageView的left、top、right、bottom这4个属性起始值和结束值。在动画执行的过程中，就可以渐渐地改变sourceImageView的这4个属性了。

下面这行代码

```
PropertyValuesHolder propertyLeft = PropertyValuesHolder.ofInt("left", mSourceImageView.getLeft(), left);
```

用**PropertyValuesHolder**可以给sourceImageView的left属性设置了起始值、结束值。

对于left属性，起始值就是sourceImageView的left值，我们已经从第一个Activity传过来了。
而left属性的结束值，我们可以从targetImageView的left属性值。
其他的top、right、bottom属性也是一样的道理。

> 需要需要特别注意的一点就是targetImageView的宽高获取方法，我们都知道获取一个View的宽高必须要等它绘制完了，而**targetImageView不会在setContentView之后立刻绘制完成**。
因此我们需要监听targetImageView的绘制状态，也就是监听**ViewTreeObserver**的各种回调，这里我们要监听的是**OnPreDrawListener**。
关于ViewTreeObserver，可以参考[《Viewtreeobserver解析》](http://souly.cn/%E6%8A%80%E6%9C%AF%E5%8D%9A%E6%96%87/2015/11/16/viewTreeObserver%E8%A7%A3%E6%9E%90/)这篇文章学习一下。

<h4 style="color:#ff0000">关键动画之ObjectAnimator - matrixAnimator实现：使ImageView展示的图片呈现渐变效果</h4>

这里的数值计算比positionAnimator要复杂一点。

首先我们要**自定义一个计算器MatrixEvaluator**，它的作用是返回动画执行过程中的Matrix，然后再使用这个Matrix去改变sourceImageView的Matrix属性。

这个自定义计算器evaluate方法非常简单，就是根据起始Matrix（startMatrix）和结束Matrix（endMatrix）之间的差值diff，然后乘以当前**加速器**返回的动画的**数值**进度即可得到当前实时的Matrix值。

下面看它的实现：

```
class MatrixEvaluator implements TypeEvaluator<Matrix> {

        public static TypeEvaluator<Matrix> NULL_MATRIX_EVALUATOR = new TypeEvaluator<Matrix>() {
            @Override
            public Matrix evaluate(float fraction, Matrix startValue, Matrix endValue) {
                return null;
            }
        };

        float[] mTempStartValues = new float[9];
        float[] mTempEndValues = new float[9];
        Matrix mTempMatrix = new Matrix();

        @Override
        public Matrix evaluate(float fraction, Matrix startValue, Matrix endValue) {
            startValue.getValues(mTempStartValues);
            endValue.getValues(mTempEndValues);
            for (int i = 0; i < 9; i++) {
                float diff = mTempEndValues[i] - mTempStartValues[i];
                // fraction是加速器中的返回值，表示当前动画的“数值”进度。我们用的是Android SDK中提供的AccelerateDecelerateInterpolator。
                mTempEndValues[i] = mTempStartValues[i] + (fraction * diff); 
            }
            mTempMatrix.setValues(mTempEndValues);

            return mTempMatrix;
        }
    }

```

有了这个计算器，得到动画执行过程中的Matrix值，怎么动态地赋给sourceImageView呢？

我们知道，在普通的`ObjectAnimator#ofFloat(Object target, String propertyName, float... values)`方法中，当Evaluator接收到最后一个可变长参数values后，可以得到起始值和结束值后。然后在**evaluate**方法中计算出动画执行过程的应该赋予的属性的值，然后调用目标对象（这里就是我们的ImageView）的setter方法把这个值赋给目标对象。

> 例如这句代码，`ObjectAnimator fadeInAnimator = ObjectAnimator.ofFloat(mContainer, "alpha", 0.0f, 1.0f);`其实就是在计算出了动画过程中每一个时刻的alpha值，然后再调用mContainer的setAlpha(float alpha)方法去改变mContainer的透明度。

但是要改变sourceImageView的Matrix值，我们需要调用**ImageView#animateTransform()**这个方法，这个方法在Android SDK中属于隐藏API，其代码片段所示：

```
// ImageView#animateTransform()源代码
/** @hide */
public void animateTransform(Matrix matrix) {
        if (mDrawable == null) {
            return;
        }
        if (matrix == null) {
            mDrawable.setBounds(0, 0, getWidth(), getHeight());
        } else {
            mDrawable.setBounds(0, 0, mDrawableWidth, mDrawableHeight);
            if (mDrawMatrix == null) {
                mDrawMatrix = new Matrix();
            }
            mDrawMatrix.set(matrix);
        }
        invalidate();
    }

```

而且这个方法的名字也不叫**setXXX**，所以我们没法调用像`ofFloat`这样的方法去改变sourceImageView的Matrix。

这时候需要采用自定义Property，并且实现它的**set**方法，自定义Property代码如下：

```
private static final Property<ImageView, Matrix> ANIMATED_TRANSFORM_PROPERTY = new Property<ImageView, Matrix>(Matrix.class, "animatedTransform") {

        @Override
        public void set(ImageView imageView, Matrix matrix) {
            // 这里模仿了SDK源码中ImageView#animateTransform的实现
            Drawable drawable = imageView.getDrawable();
            if (drawable == null) {
                return;
            }
            if (matrix == null) {
                drawable.setBounds(0, 0, imageView.getWidth(), imageView.getHeight());
            } else {
                drawable.setBounds(0, 0, drawable.getIntrinsicWidth(), drawable.getIntrinsicHeight());
                Matrix drawMatrix = imageView.getImageMatrix();
                if (drawMatrix == null) {
                    drawMatrix = new Matrix();
                    imageView.setImageMatrix(drawMatrix);
                }
                imageView.setImageMatrix(matrix);
            }
            imageView.invalidate();
        }

        @Override
        public Matrix get(ImageView object) {
            return null;
        }
    };
```

自定义一个Property必须要实现里面的get方法，但是在我们的这里例子中，get方法不会被调用。

因为在属性动画中，只有当你传入的可变长参数values（也就是起始值、中间值1、中间值2. ... 结束值）长度为1，也就是说你只传了一个值的时候，才会对我们的target调用getter方法去获取初始值。所以这里我们是不需要getter方法的。

自定义Property完成后，通过`ObjectAnimator#ofObject(T target, Property<T, V> property, TypeEvaluator<V> evaluator, V... values)` 方法，就可以把计算器计算出的动画执行过程中的Matrix值，通过自定义Property中的set方法，赋给当前的目标对象，即sourceImageView！从而使得sourceImageView呈现出渐变效果。

> 这里的实现是来自这位大神的博客[Implementing ImageView transition between activities for pre-Lollipop devices](https://medium.com/@v.danylo/implementing-imageview-transition-between-activities-for-pre-lollipop-devices-8b24bc387a2a#.7c6qxvf59)。
我们首先感谢这位大神的分享！这位大神在文中也提到，用动画来实现图片的渐进式改变，起实来自于我们Android SDK中的隐藏API——**ImageView#animateTransform**。

## 2.退场

退场动画完全是入场动画的逆操作，直接看代码。

```
    // 图片退出的转场动画：完全是和之前相反的过程
    private void playExitAnimations(int sourceImageViewLeft, int sourceImageViewTop, int sourceImageViewWidth, int sourceImageViewHeight, float[] imageMatrixValues) {
        mSourceImageView.setVisibility(View.VISIBLE);
        mTargetImageView.setVisibility(View.INVISIBLE);

        // 改变SourceView的位置、宽高属性。这里每个属性的起始值和结束值和入场时刚好相反。
        int[] locationOnScreen = new int[2];
        mSourceImageView.getLocationOnScreen(locationOnScreen);
        PropertyValuesHolder propertyLeft = PropertyValuesHolder.ofInt("left", locationOnScreen[0], sourceImageViewLeft);
        PropertyValuesHolder propertyTop = PropertyValuesHolder.ofInt("top", locationOnScreen[1], sourceImageViewTop);
        PropertyValuesHolder propertyRight = PropertyValuesHolder.ofInt("right", locationOnScreen[0] + mSourceImageView.getWidth(), sourceImageViewLeft + sourceImageViewWidth);
        PropertyValuesHolder propertyBottom = PropertyValuesHolder.ofInt("bottom", mSourceImageView.getBottom(), sourceImageViewTop + sourceImageViewHeight);
        ObjectAnimator positionAnimator = ObjectAnimator.ofPropertyValuesHolder(mSourceImageView, propertyLeft, propertyTop, propertyRight, propertyBottom);

        // ImageView的矩阵动画
        Matrix initialMatrix = getImageMatrix(mSourceImageView);

        Matrix endMatrix = new Matrix();
        endMatrix.setValues(imageMatrixValues);
        mSourceImageView.setScaleType(ImageView.ScaleType.MATRIX);
        // 这里Matrix的起始值和结束值和入场时也刚好相反。
        ObjectAnimator matrixAnimator = ObjectAnimator.ofObject(mSourceImageView, ANIMATED_TRANSFORM_PROPERTY, new MatrixEvaluator(), initialMatrix, endMatrix);

        // 渐变动画
        ObjectAnimator fadeInAnimator = ObjectAnimator.ofFloat(mContainer, "alpha", 1.0f, 0.0f);

        mExitingAnimation = new AnimatorSet();
        mExitingAnimation.setDuration(IMAGE_TRANSLATION_DURATION);
        mExitingAnimation.setInterpolator(new AccelerateInterpolator());
        mExitingAnimation.addListener(new Animator.AnimatorListener() {

            @Override
            public void onAnimationStart(Animator animation) {

            }

            @Override
            public void onAnimationEnd(Animator animation) {
                if (mExitingAnimation != null) {
                    mExitingAnimation = null;
                }
                // 关闭第二个界面
                Activity activity = (Activity) mSourceImageView.getContext();
                activity.finish();
                activity.overridePendingTransition(0, 0); // 同样去掉默认的转场动画
            }

            @Override
            public void onAnimationCancel(Animator animation) {

            }

            @Override
            public void onAnimationRepeat(Animator animation) {

            }
        });

        mExitingAnimation.playTogether(positionAnimator, matrixAnimator, fadeInAnimator);
        mExitingAnimation.start();
    }
```

# 四、涉及到的知识点

我认为实现demo里面的效果需要了解下面的知识点，如果不熟悉的话，建议先看一下上一章推荐的属性动画讲解的几篇博客。

## 1.ImageView的ScaleType

不管将ScaleType设为多少，bitmap始终都是一个。如果在Android Studio打开debug模式来查看bitmap实际的图片，用一个ImageView去展示一张图片，不管你怎么改变ScaleType，其实里面的图片对象都是一样的。

## 2.属性动画之插值器 - Interpolator

控制动画数值进度的转换器，我们给动画是指一个duration之后，插值器负责把动画的**自然**进度转成**数值**进度。自然进度就是指随着时间匀速增长的值。

所有的插值器都实现了**TimeInterpolator接口里面的public float getInterpolation(float input)**方法，input就是随时间流逝的自然进度，在这个方法中根据实际需求，用input计算出实际数字，作为数值进度返回。

## 3.属性动画之计算器 - TypeEvaluator

计算器就是计算动画执行过程中，目标对象的某个属性的数值。

TypeEvaluator接口中有一个**public T evaluate(float fraction, T startValue, T endValue)**方法，fraction就是插值器返回的数值进度，而startValue就是对象的某一个属性的起始值，endValue是这个属性的结束值。

这里利用的是泛型编程，我们可以把属性的起始、结束值看成一个Type。传入自己定义的任何Type后，在evaluate方法中，计算出当前应该改变**对象**的**属性**的具体Type值。再调用这个对象的setter方法，将Type值赋给这个对象。

很多时候，我们不会像这个demo中自定义Property，然后把它set给一个系统封装好的ImageView。我们很可能会有一个自定义的CustomView，然后在这个CustomView中提供一个setXXX方法。这样也可以在自定义的计算器中实现CustomView的属性动态改变。

## 4.属性动画之中的ObjectAnimator和ValueAnimator的区别

ObjectAnimator是ValueAnimator的子类，ValueAnimator只负责计算动画过程中，目标对象（一般是一个View或者其他UI元素）属性的值，但是需要我们自己监听动画的update状态，再把监听到的值set给目标对象的属性。

ObjectAnimator除了可以计算动画过程中的属性值外，还可以调用目标对象的setter方法，改变这个属性的值。所以它的功能比ValueAnimator要强大。

## 4.属性动画之PropertyValuesHolder用法

一般直接使用`ObjectAnimator ofFloat(Object target, String propertyName, float... values)`只能改变目标对象的一个属性值。

如果我们想要改一个目标对象的的多个属性时，可以先使用`PropertyValuesHolder ofInt(String propertyName, int... values)`创建PropertyValuesHolder。

然后再用`ObjectAnimator ofPropertyValuesHolder(Object target, PropertyValuesHolder... values) `创建出改变多个属性的属性动画对象ObjectAnimator。

## 5.如何向ContentView中动态添加View

首先要从当前的Activity中获得根视图：

```
getWindow().getDecorView().findViewById(android.R.id.content);
```

这是个FrameLayout，然后我们就可以用java代码动态向它里面添加sourceImageView了。

关于DecorView再多说两句，它是Activity界面的根View，继承自FrameLayout。在它里面又是一个LinearLayout，在这个LinearLayout里面又包含了**id为@android:id/title_container**的标题栏，和一个**id为@android:id/content**的ContentView，结构大概是下面这个样子的：

```
- DecorView
    - LinearLayout
        - ...
        - FrameLayout：android:id/title_container
        - FrameLayout：@android:id/content
```

当我们在onCreate方法中调用Activity#setContentView()时，会把我们自己写的布局添加到这个ContentView中去。

# 五、Android 5.0上的实现方法

下面是在Android 5.0以上一种示例，非常简单，只需要几行代码就可实现：

```
    // 第一个Activity，利用ActivityOptions创建SceneTransitionAnimation
    private void transitionOnAndroidL() {
        // 把需要共享的元素-ImageView，传给第二个界面
        Intent intent = new Intent(MainActivity.this, DetailActivityLollipop.class);
        // 一定要传入shareElementName
        String shareElementName = "sharedImageView";
        ActivityOptions activityOptions = ActivityOptions.makeSceneTransitionAnimation(this, mImageView, shareElementName);
        getWindow().setSharedElementEnterTransition(new ChangeImageTransform(this, null));
        intent.putExtra(DetailActivityLollipop.SHARED_ELEMENT_KEY, shareElementName);
        intent.putExtra(DetailActivityLollipop.IMAGE_RES_ID, mImageResId);
        // 打开它
        startActivity(intent, activityOptions.toBundle());
    }

    // 第二个Activity，取出shareElementName，再调用ViewCompat#setTransitionName
    private void initImageEnterTransition() {
        imageView.setVisibility(View.VISIBLE);
        String imageTransitionName = getIntent().getStringExtra(SHARED_ELEMENT_KEY);
        ViewCompat.setTransitionName(imageView, imageTransitionName);

        View mainContainer = findViewById(R.id.activityContanierDetail);
        mainContainer.setAlpha(1.0f);
        int resId = getIntent().getExtras().getInt(IMAGE_RES_ID);
        imageView.setImageResource(resId);
    }


```

官方介绍的[Transitions](https://developer.android.com/training/material/animations.html#Transitions)中用xml也可以实现。另外，还有多个元素的转场动画效果，这里就不详细说了，如果有需要，也可以参考下这篇文章——[Shared Element Activity Transition](https://guides.codepath.com/android/Shared-Element-Activity-Transition)。

# 六、其他实现方法

在我的demo中只演示了核心的View转场实现，没有和其他的稍微复杂一些的需求相结合。网上还有很多关于这种效果实现的分享，也有应用到一些更复杂场景，下面推荐出来一起多多学习。

文章推荐：

- [Android中的转场动画及兼容处理](http://wl9739.github.io/2016/10/16/Android-%E4%B8%AD%E7%9A%84%E8%BD%AC%E5%9C%BA%E5%8A%A8%E7%94%BB%E5%8F%8A%E5%85%BC%E5%AE%B9%E5%A4%84%E7%90%86/)

- [Android共享元素转场动画兼容实践](http://www.jianshu.com/p/340c938e9f32)

- [两步实现类似格瓦拉的转场动画](http://immortalz.me/859.html)

- [Shared Element Activity Transition](https://guides.codepath.com/android/Shared-Element-Activity-Transition)：详细介绍了5.0以上的各种共享元素转场效果。

开源Library推荐：

- [PreLollipopTransition](https://github.com/takahirom/PreLollipopTransition)

- [ImageTransition](https://github.com/vikramkakkar/ImageTransition)

- [GestureViews](https://github.com/alexvasilkov/GestureViews)：手势操作库，其demo本身就实现了一个类似的转场的动画效果。

