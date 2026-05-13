### 一级缓存：屏幕内缓存（mAttachedScrap）

屏幕内缓存指在屏幕中显示的ViewHolder，这些ViewHolder会缓存在mAttachedScrap、mChangedScrap中 ：

- mChangedScrap 表示数据已经改变的ViewHolder列表，需要重新绑定数据（调用onBindViewHolder）
- mAttachedScrap 未与RecyclerView分离的ViewHolder列表

### 二级缓存：屏幕外缓存（mCachedViews）

用来缓存移除屏幕之外的 ViewHolder，默认情况下缓存容量是 2，可以通过 setViewCacheSize 方法来改变缓存的容量大小。如果 mCachedViews 的容量已满，则会优先移除旧 ViewHolder，把旧ViewHolder移入到缓存池RecycledViewPool 中。

### 三级缓存：自定义缓存（ViewCacheExtension）

给用户的自定义扩展缓存，需要用户自己管理 View 的创建和缓存，可通过Recyclerview.setViewCacheExtension()设置。

### 四级缓存：缓存池（RecycledViewPool ）

ViewHolder 缓存池，在mCachedViews中如果缓存已满的时候（默认最大值为2个），先把mCachedViews中旧的ViewHolder 存入到RecyclerViewPool。如果RecyclerViewPool缓存池已满，就不会再缓存。从缓存池中取出的ViewHolder ，需要重新调用bindViewHolder绑定数据。