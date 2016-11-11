---
title: 如何利用RecyclerView打造炫酷滑动卡片
date: 2016-11-11 10:29:46
tags:
---

## 前言

前段时间一直在B站追《黑镜》第三季，相比前几季，这季很良心的拍了六集，😄着实过了一把瘾。由于看的是字幕组贡献的版本，每集开头都插了一个app的广告，叫“人人美剧”，一向喜欢看美剧的我便扫了一下二维码，安装了试一试。我打开app，匆匆滑动了一下首页的美剧列表，然后便随手切换到了订阅页面，然后，我就被订阅页面的动画效果吸引住了。

<!-- more -->


![](http://od35ecbnl.bkt.clouddn.com/renrenmeiju.gif)

没错，就是上面这玩意儿，是不是很炫酷，本着发扬一名码农的职业精神，我心里便痒痒的想实现这种效果，当然因为长期的fork compile，第一时间我还是上网搜了搜，有木有哪位好心人已经开源了类似的控件。借助强大的Google，我马上搜到了一个项目 [SwipeCards](https://github.com/Diolor/Swipecards)，是仿照探探的老父亲Tinder的app动画效果打造的，果然程序员都一个操行，看到好看的就想动手实现,不过人家的成绩让我可望而不可及~

![](http://od35ecbnl.bkt.clouddn.com/SwipeCards.png)

他实现的效果是这样的：

![](http://od35ecbnl.bkt.clouddn.com/SwipeCardsAnim.gif)

嗯，还不错，为了进行思想上的碰撞，我就download了一下他的源码，稍稍read了一下~_~

作为一个有思想，有抱负的程序员，怎么能满足于compile别人的库呢？必须得自己动手，丰衣足食啊！

## 开工

###思考

一般这种View都是自定义的，然后重写onLayout，但是有木有更简单的方法呢？由于项目里一直使用RecyclerView，那么能不能用RecyclerView来实现这种效果呢？能，当然能啊！得力于RecyclerView优雅的扩展性，我们完全可以自定义一个LayoutManager来实现嘛。

### 布局实现

RecyclerView可以通过自定义LayoutManager来实现各种布局，官方自己提供了LinearLayoutManager、GridLayoutManager，相比于ListView，可谓是方便了不少。同样，我们也可以通过自定义LayoutManager，实现这种View一层层叠加的效果。

自定义LayoutManager，最重要的是要重写onLayoutChildren()

    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        detachAndScrapAttachedViews(recycler);
        for (int i = 0; i < getItemCount(); i++) {
            View child = recycler.getViewForPosition(i);
            measureChildWithMargins(child, 0, 0);
            addView(child);
            int width = getDecoratedMeasuredWidth(child);
            int height = getDecoratedMeasuredHeight(child);
            layoutDecorated(child, 0, 0, width, height);
            if (i < getItemCount() - 1) {
                child.setScaleX(0.8f);
                child.setScaleY(0.8f);
            }
        }
    }

这种布局实现起来其实相当简单，因为每个child的left和top都一样，直接设置为0就可以了，这样child就依次叠加在一起了，至于最后两句，主要是为了使顶部Child之下的childs有一种缩放的效果。

### 动画实现

下面到了最重要的地方了，主要分为以下几个部分。

#### (1)手势追踪

当手指按下时，我们需要取到RecyclerView的顶部Child，并让其跟随手指滑动。

    public boolean onTouchEvent(MotionEvent e) {
        if (getChildCount() == 0) {
            return super.onTouchEvent(e);
        }
        View topView = getChildAt(getChildCount() - 1);
        float touchX = e.getX();
        float touchY = e.getY();
        switch (e.getAction()) {
            case MotionEvent.ACTION_DOWN:
                mTopViewX = topView.getX();
                mTopViewY = topView.getY();
                mTouchDownX = touchX;
                mTouchDownY = touchY;
                break;
            case MotionEvent.ACTION_MOVE:
                float dx = touchX - mTouchDownX;
                float dy = touchY - mTouchDownY;
                topView.setX(mTopViewX + dx);
                topView.setY(mTopViewY + dy);
                updateNextItem(Math.abs(topView.getX() - mTopViewX) * 0.2 / mBorder + 0.8);
                break;
            case MotionEvent.ACTION_UP:
                mTouchDownX = 0;
                mTouchDownY = 0;
                touchUp(topView);
                break;
        }
        return super.onTouchEvent(e);
    }

手指按下的时候，记录topChildView的位置，移动的时候，根据偏移量，动态调整topChildView的位置，就实现了基本效果。但是这样还不够，记得我们在实现布局时，对其他子View进行了缩放吗？那时候的缩放是为现在做准备的。当手指在屏幕上滑动时，我们同样会调用updateNextItem()，对topChildView下面的子view进行缩放。

    private void updateNextItem(double factor) {
        if (getChildCount() < 2) {
            return;
        }
        if (factor > 1) {
            factor = 1;
        }
        View nextView = getChildAt(getChildCount() - 2);
        nextView.setScaleX((float) factor);
        nextView.setScaleY((float) factor);
    }

这里的factor计算很简单，只要当topChildView滑动到设置的边界时，nextView刚好缩放到原本大小，即factor=1，就可以了。因为nextView一开始缩放为0.8，所以可计算出：

	factor=Math.abs(topView.getX() - mTopViewX) * 0.2 / mBorder + 0.8

#### (2)抬起手指

手指抬起后，我们要进行状态判断

**1.滑动未超过边界**

此时我们需要对topChildView进行归位。

**2.超过边界**

此时我们需要根据滑动方向，使topChildView飞离屏幕。

对于这两种情况，我们都是通过计算view的终点坐标，然后利用动画实现的。对于第一种，很简单，targetX和targetY直接就是topChildView的原始坐标。但是对于第二种，需要根据topChildView的原始坐标和目前坐标，计算出线性表达式，然后再根据targetX来计算targetY，至于targetX，往右飞targetX就可以赋为getScreenWidth，而往左就直接为0-view.width，只要终点在屏幕外就可以。具体代码如下。

    private void touchUp(final View view) {
        float targetX = 0;
        float targetY = 0;
        boolean del = false;
        if (Math.abs(view.getX() - mTopViewX) < mBorder) {
            targetX = mTopViewX;
            targetY = mTopViewY;
        } else if (view.getX() - mTopViewX > mBorder) {
            del = true;
            targetX = getScreenWidth()*2;
            mRemovedListener.onRightRemoved();
        } else {
            del = true;
            targetX = -view.getWidth()-getScreenWidth();
            mRemovedListener.onLeftRemoved();
        }
        View animView = view;
        TimeInterpolator interpolator = null;
        if (del) {
            animView = getMirrorView(view);
            float offsetX = getX() - mDecorView.getX();
            float offsetY = getY() - mDecorView.getY();
            targetY = caculateExitY(mTopViewX + offsetX, mTopViewY + offsetY, animView.getX(), animView.getY(), targetX);
            interpolator = new LinearInterpolator();
        } else {
            interpolator = new OvershootInterpolator();
        }
        final boolean finalDel = del;
        animView.animate()
                .setDuration(500)
                .x(targetX)
                .y(targetY)
                .setInterpolator(interpolator)
                .setUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
                    @Override
                    public void onAnimationUpdate(ValueAnimator animation) {
                        if (!finalDel) {
                            updateNextItem(Math.abs(view.getX() - mTopViewX) * 0.2 / mBorder + 0.8);
                        }
                    }
                });

    }
    
对于第二种情况，如果直接启动动画，并在动画结束时通知adapter删除item，在连续操作时，会导致数据错乱。但是如果在动画启动时直接移除item，又会失去动画效果。所以我在这里采用了另一种办法，在动画开始前创建一个与topChildView一模一样的镜像View，添加到DecorView上，并隐藏删除掉topChildView，然后利用镜像View来展示动画。添加镜像View的代码如下：

    private ImageView getMirrorView(View view) {
        view.destroyDrawingCache();
        view.setDrawingCacheEnabled(true);
        final ImageView mirrorView = new ImageView(getContext());
        Bitmap bitmap = Bitmap.createBitmap(view.getDrawingCache());
        mirrorView.setImageBitmap(bitmap);
        view.setDrawingCacheEnabled(false);
        FrameLayout.LayoutParams params = new FrameLayout.LayoutParams(bitmap.getWidth(), bitmap.getHeight());
        int[] locations = new int[2];
        view.getLocationOnScreen(locations);

        mirrorView.setAlpha(view.getAlpha());
        view.setVisibility(GONE);
        ((SwipeCardAdapter) getAdapter()).delTopItem();
        mirrorView.setX(locations[0] - mDecorViewLocation[0]);
        mirrorView.setY(locations[1] - mDecorViewLocation[1]);
        mDecorView.addView(mirrorView, params);
        return mirrorView;
    }
    
因为镜像View是添加在DecorView上的，topChildView父容器是RecyclerVIew，而View的x、y是相对于父容器而言的，所以镜像View的targetX和targetY需要加上一定偏移量。

好了到这里，一切就准备就绪了，下面让我们看看动画效果如何。

![](http://od35ecbnl.bkt.clouddn.com/swipecard.gif)

## 总结
效果是不是还不错，项目地址在这里[https://github.com/HalfStackDeveloper/SwipeCardRecyclerView](https://github.com/HalfStackDeveloper/SwipeCardRecyclerView)，欢迎大家fork AND star！也希望大家在使用app，看到一些酷炫效果的时候，也自己去动手实现，谁让我们是有着职业精神的码农呢！

**(转载请标明ID：半栈工程师，个人博客：[https://halfstackdeveloper.github.io](https://halfstackdeveloper.github.io))**