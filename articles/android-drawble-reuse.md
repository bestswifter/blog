# 复用 Drawable

**Drawable** 表示了一类通用的，可被绘制的资源。与View 的主要区别就在于，Drawable 并不会响应事件。

## Drawable 的复用机制

Drawable 的使用非常方便，系统框架内部有 700 多种默认的 Drawable。当我们新建一个 button 时，实际上它的背景就是一个默认的 Drawable。

每个 Drawable 都有一个 `constant state`，这个 state 中保存了Drawable 所有的关键信息。比如对于 Button 来说，其中就保存了用于展示的 Bitmap。

由于 Drawable 非常常用，为了优化性能（其实主要就是节省内存），所有的 Drawable 都共享同一个 `constant state`。

## 重用 state

这种优化有时候也会导致一些问题，比如：

```java
Drawable star = context.getResources().getDrawable(R.drawable.star);
if (book.isFavorite()) {
  star.setAlpha(255); // opaque
} else {
  star.setAlpha(70); // translucent
}
```

假设有多个 book 对象构成一个 `listview`，我们希望的效果是喜欢的图书，星星是亮的，否则是灭的。但如果使用上述代码就会发现，所有星星的颜色都是一样的。

这是因为 alpha 信息保存在 constant state 中，所有的星星都共享这个 state，对任何一个的修改都会影响其他所有的。

解决方案是使用 `mutate` 方法。这个方法会返回同一个 Drawable 对象，但是其中的 state 被复制了，这样对 state 的修改就互不干扰：

```java
Drawable star = context.getResources().getDrawable(R.drawable.star);
if (book.isFavorite()) {
  star.mutate().setAlpha(255); // opaque
} else {
  star.mutate().setAlpha(70); // translucent
}
```

## 重用 bitmap

不过有时候仅仅复制 state 还不够，因为所有的 state 还会共享同一个 Bitmap，也就是说调用 `mutate()` 方法并不会复制 Bitmap。

假设我们有两个 TextView 需要设置圆角，我们可以首先创建一个 `GradientDrawable` 对象并设置圆角：

```java
GradientDrawable gd = new GradientDrawable();
gd.setColor(Color.parseColor("#000000"));
gd.setCornerRadius(context.getResources().getDimension(R.dimen.ds4));
gd.setStroke(1, Color.parseColor("#000000"));
```

接下来，任何需要设置圆角背景的 TextView 都可以调用：

```java
textview.setBackgroundDrawable(gd);
```

然而由于 Drawable 对象的 Bitmap 会被复用，所以即使我们调用了 `mutate()` 方法，所有的 TextView 的圆角背景区域依然都会以最后一个 TextView 的大小为准。

在这种情况下，我们可以通过 `constant state` 创建一个新的 Drawable 对象，此时这两个完全不同的对象会使用不用的 Bitmap，也就避免了上述问题：

```java
textview.setBackgroundDrawable(gd.getConstantState().newDrawable());
```
