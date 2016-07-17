---
title: RecyclerView和ListView的比较
date: 2016-07-18 00:49:30
tags:
---

# RecyclerView和ListView使用上的比较

| 对比项     | 区别（ListView）                             | 区别（RecyclerView）                         |
| ------- | ---------------------------------------- | ---------------------------------------- |
| Adapter | 自己实现ViewHolder                           | RecyclerView.Adapter处理数据集合并负责绑定视图，自带ViewHolder（必须） |
| 动画      | 实现麻烦                                     | ItemAnimator                             |
| 布局      | 只能实现垂直线性排列的列表视图                          | LayoutManager，可以左右滑，瀑布流（必须）              |
| 分割线     | android:divider                          | ItemDecoration，更灵活多变                     |
| 点击事件    | AdapterView.OnItemClickListener    AdapterView.OnItemLongClickListener | 只能自定义，可用RcyclerView.OnItemTouchListener来响应触摸事件 |

> 可以把RecyclerView理解为ListView更高一层的抽象

# ListView

## ListView的通用写法

``` Java
lv.setAdapter(new BaseAdapter() {
            @Override
            public int getCount() {
                return dataList.size();
            }

            @Override
            public Object getItem(int i) {
                return dataList.get(i);
            }

            @Override
            public long getItemId(int i) {
                return i;
            }

            @Override
            public View getView(int i, View view, ViewGroup viewGroup) {
                ViewHolder vh = new ViewHolder();
                if (view == null) {
                    view=LayoutInflater.from(MainActivity.this).inflate(R.layout.itemlayout, viewGroup, false);
                    vh.tv= (TextView) view.findViewById(R.id.tv);
                    view.setTag(vh);
                }else {
                    vh= (ViewHolder) view.getTag();
                }

                vh.tv.setText(getItem(i).toString());

                return view;
            }

            class ViewHolder {
                TextView tv;
            }
        });
```

getView()完成了两件事：1.View的生成（如果没有缓存的话）2.数据的绑定

## 为什么这样写？

### ViewHolder的作用

findViewById会遍历子View，耗时，使用ViewHolder可以将遍历的结果（itemview的子view的引用）缓存下来下次直接使用。（利用View的tag来缓存item view子view的引用）

### 使用getView()第二个参数

第二个参数会从缓存中取出一个缓存的View，如果不存在则创建它，因为使用LayoutInflate创建View需要解析xml文件，这比较耗时。（利用ListView的回收机制来缓存View）

##源码分析

### 继承树：

ListView <- AbsListView <- AdapterView <- ViewGroup <- View

### notifyDataSetChanged()的秘密

典型的**观察者模式**

BaseAdapter里面有个DataSetObservable类型的对象，里面保存了DataSetObserver的列表（`在list.setAdapter()`的时候加进去的），DataSetObserver在AdapterView中有一个内部实现类AdapterDataSetObserver，并且在AbsListView中有一个实例mDataSetObserver。当调用`adapter.notifyDataSetChanged()`的时候，就会遍历DataSetObserver列表，通知到对应的ListView，完成刷新。（Adapter是通知者，ListView是观察者）

### 回收机制

RecycleBin类，是AbsListView的一个内部类，其作用是根据不同的view type缓存view，ListView缓存机制的实现者。

``` java
private ArrayList<View>[] mScrapViews;
```

缓存view的主要数据结构，数组的长度为view type的count，数组的每一项为一个ArrayList，里面缓存了对应的view，由此可知不同type的view是分开缓存的。

```java
void addScrapView(View scrap, int position)
```

用来收集废弃的view，还会在其中回调mRecyclerListener.onMovedToScrapHeap()，比如说设置的动画如果还没有完成需要在回调里结束掉。

```java
View getScrapView(int position)
```

用来从指定位置获取一个缓存的view，内部会先计算出指定位置的type，再从mScrapViews中对应的type里取出一个缓存view，再将该缓存view从mScrapViews中删除。

```java
View obtainView(int position, boolean[] isScrap)
```

位于AbsListView，作用是获得指定位置已经绑定数据的View，其内部会从缓存中获取View，并作为第二个参数来调用adapter.getView()


```java
private void scrollListItemsBy(int amount)
```

位于ListView中，在向下滑动过程中，上端是即将出现新view的一端，调用obtainView()获得一个绑定了数据的view，然后将其加入上端，当下端的itemview移出可视范围后调用recycleBin.addScrapView()将其移入缓存。

# RecyclerView 

## 缓存的异同

ListView缓存的是View，RecyclerView缓存的是ViewHolder，优点：将ViewHolder强制加入缓存机制中，省去了从缓存的View中获取ViewHolder的步骤。

## 源码分析

### 继承树：

RecyclerView <- ViewGroup

### 回收机制：

RecycledViewPool类，位于RecyclerView中，类似ListView中的RecycleBin类

```java
private SparseArray<ArrayList<ViewHolder>> mScrap =
        new SparseArray<ArrayList<ViewHolder>>();
```

类似RecycleBin类中的mScrapViews，只不过缓存的对象是ViewHolder。也是根据viewType来缓存的，其中SparseArray可以看做是一个优化过的ArrayList。

ViewCacheExtension类，是一个抽象类，只有一个抽象方法

``` java
abstract public View getViewForPositionAndType(Recycler recycler, int position, int type);
```

Recycler类，作为回收机制的统一管理者，多级缓存管理

1. 检查`mChangedScrap`，若匹配到则返回相应holder 
2. 检查`mAttachedScrap`，若匹配到且holder有效则返回相应holder 
3. 检查`mViewCacheExtension`，若匹配到则返回相应holder 
4. 检查`mRecyclerPool`，若匹配到则返回相应holder 
5. 否则执行`Adapter.createViewHolder()`，新建holder实例
6. 返回holder.itemView 

注：以上每步匹配过程都可以匹配position或itemId（如果有stableId）

## LayoutManager

`LayoutManager`主要作用是，测量和摆放RecyclerView中itemView，以及当itemView对用户不可见时循环复用处理。 通过设置Layout Manager的属性，可以实现水平滚动、垂直滚动、方块表格等列表形式。其内部类`Properties`包含了所需要的大部分属性，使用详情见参考资料。

## ViewHolder

ViewHolder在RecyclerView中成了必选项，而且功能更强大，除了能够持有itemview的子view的引用外，还有很多基本信息

``` java
public static abstract class ViewHolder {
    View itemView;//itemView
    int mPosition;//位置
    int mOldPosition;//上一次的位置
    long mItemId;//itemId
    int mItemViewType;//itemViewType
    int mPreLayoutPosition;
    int mFlags;//ViewHolder的状态标志
    int mIsRecyclableCount;
    Recycler mScrapContainer;//若非空，表明当前ViewHolder对应的itemView可以复用
｝
```

## ItemDecoration

``` java
public static abstract class ItemDecoration {
    //itemView绘制之前绘制，通常这部分UI会被itemView盖住
    public void onDraw(Canvas c, RecyclerView parent, State state) {
        onDraw(c, parent);
    }

    //itemView绘制之后绘制，这部分UI盖在itemView上面
    public void onDrawOver(Canvas c, RecyclerView parent, State state) {
        onDrawOver(c, parent);
    }

    //设置itemView上下左右的间距
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state) {
        getItemOffsets(outRect, ((LayoutParams) view.getLayoutParams()).getViewLayoutPosition(),
                parent);
    }
}
```

## ItemAnimator

将itemview的动画统一管理的一个类，可以自定义itemview添加，移除，移动的动画，具体使用见参考资料。

## 局部刷新

比起ListView的全局刷新adapter.notifyDataSetChanged()，RecyclerView的Adapter有更多的刷新方法，可以针对指定位置的添加，删除，移动操作进行刷新操作，RecyclerView.AdapterDataObserver类比ListView的DataSetObservable也拥有更多的回调。

```java
/**
 * Observer base class for watching changes to an {@link Adapter}.
 * See {@link Adapter#registerAdapterDataObserver(AdapterDataObserver)}.
 */
public static abstract class AdapterDataObserver {
    public void onChanged() {
        // Do nothing
    }

    public void onItemRangeChanged(int positionStart, int itemCount) {
        // do nothing
    }

    public void onItemRangeChanged(int positionStart, int itemCount, Object payload) {
        // fallback to onItemRangeChanged(positionStart, itemCount) if app
        // does not override this method.
        onItemRangeChanged(positionStart, itemCount);
    }

    public void onItemRangeInserted(int positionStart, int itemCount) {
        // do nothing
    }

    public void onItemRangeRemoved(int positionStart, int itemCount) {
        // do nothing
    }

    public void onItemRangeMoved(int fromPosition, int toPosition, int itemCount) {
        // do nothing
    }
}
```

# 参考资料

[Android AdapterView 源码分析以及其相关回收机制的分析]: http://www.cnblogs.com/youxilua/archive/2013/05/11/3072353.html
[深入理解 RecyclerView 系列之一：ItemDecoration]: https://blog.piasy.com/2016/03/26/Insight-Android-RecyclerView-ItemDecoration/
[RecyclerView源码分析]: http://mouxuejie.com//blog/2016-03-06/recyclerview-analysis/
[Android RecyclerView使用及自定义itemAnimator]: http://coderrobin.com/2015/02/11/Android-RecyclerView%E4%BD%BF%E7%94%A8/
[创建一个 RecyclerView LayoutManager – Part 1]: http://www.devtf.cn/?p=351

