

## 安卓Feed流方案实现探讨  - 元文兵



### 安卓Feed流方案实现探讨（基于RecyclerView）



#### 引子

Feed流开发，经常要做的的工作的就是增加新的itemType，每增加一种itemType，都需要修改一些逻辑来，来让列表支持新添加的类型。

#### 普通实现方式

要实现必须重写和实现以下方法：

##### 通过数据返回需要显示的类型

```java
@Override
public int getItemViewType(int position) {
    if (mDataList.get(position).getType().equals("0")) {
        return 0;
    } else if (mDataList.get(position).getType().equals("1")) {
        return 1;
    }
    return 0;
}
```



##### 根据不同的类型创建ViewHolder处理itemView初始化

```java
@NonNull
@Override
public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
    if (viewType == 0) {
         View item0View = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_layout0, parent, false);
         return new ItemViewHolder0(item0View);
     } else if (viewType == 1) {
         View item1View = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_layout1, parent, false);
         return new ItemViewHolder1(item1View);
     }
      View item0View = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_layout0, parent, false);
      return new ItemViewHolder0(item0View);
    }
```



##### 创建不同的ViewHolder

```java
static class ItemViewHolder0 extends RecyclerView.ViewHolder {
        private final TextView item0Tv;

        public ItemViewHolder0(@NonNull View itemView) {
            super(itemView);
            item0Tv = itemView.findViewById(R.id.item0_tv);
        }
    }

static class ItemViewHolder1 extends RecyclerView.ViewHolder {
        private final TextView item1Tv;

        public ItemViewHolder1(@NonNull View itemView) {
            super(itemView);
            item1Tv = itemView.findViewById(R.id.item1_tv);
        }
    }
```



##### 绑定数据

```java
 @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position) {
	      // 设置数据
        if (holder instanceof ItemViewHolder0) {
           	
        } else if (holder instanceof ItemViewHolder1) {
          
        }

    }
```



从上面的代码可以看出，增加一种类型的成本是巨大的，上例中还仅仅是添加简单itemType类型，如果达到数十种类型，工作量无疑是庞大的，if else 也会增加很多，所有的类型都会在这一个类中编写，并且需要多个ViewHolder来创建和缓存View,Adapter就会越来越庞大。

当然如果只几种类型的列表时，是可以使用这种方式的。



#### 安卓财经使用方式(延用张鸿洋的一种写法）

简单使用如下：

##### adapter要继承`MultiItemTypeAdapter`

```java
public class NewsFeedListAdapter extends MultiItemTypeAdapter {

    public NewsFeedListAdapter(Context context, List datas, ZiXunType type) {
        super(context, datas);
        initDelegate(context, type);
    }

    private void initDelegate(Context context, ZiXunType type) {
        /**
         * 新闻类型一张图,没有图,广告一张图正常模式,长标题模式,直播,专题
         */
        addItemViewDelegate(new NewsFeedOneImgViewDelegate(isHasFeedback));

        /**
         *  新闻类型三张图,广告类型三张图
         */
        addItemViewDelegate(new NewsFeedThreeImgViewDelegate(isHasFeedback));

        /**
         * 广告类型大图
         */
        addItemViewDelegate(new NewsFeedBigImgAdViewDelegate());
      
	      .....
}
```

##### itemView要继承自`ItemViewDelegate`，通过`getItemViewLayoutId`提供xml布局文件，在`covert`方法里面进行view的赋值和展示逻辑操作。

```java
public class NewsFeedOneImgViewDelegate implements ItemViewDelegate {

    @Override
    public int getItemViewLayoutId() {
        return R.layout.listitem_feed_news_oneimg_layout;
    }

    @Override
    public boolean isForViewType(Object object, int position) {
				if ((object instanceof TYFeedItem)) {//新闻与图片数量为1
            TYFeedItem item = (TYFeedItem) object;
            if (item.getType() == TYFeedItem.NEW_TYPE_VIDEO && FinanceApp.getInstance().noPictureModeOpen) {
                return true;
            }
            return item != null && getImgList(item).size() <= 2 && !item.isExcludeType();
        }
        return false;
    }

    @Override
    public void convert(final ViewHolder holder, Object object, final int position){
    		//展示数据
		}
}
```

##### 再看一下MultiItemTypeAdapter`：

```java
public class MultiItemTypeAdapter<T> extends RecyclerView.Adapter<ViewHolder> {

    public MultiItemTypeAdapter(Context context, List<T> datas) {
        if(!(this instanceof CommonAdapter)) {
            //托底
            addItemViewDelegate(new EmptyItemViewDelegate());
        }
    }

    @Override
    public int getItemViewType(int position) {
        return mItemViewDelegateManager.getItemViewType(mDatas.get(position), position);
    }


    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        ItemViewDelegate itemViewDelegate = mItemViewDelegateManager.getItemViewDelegate(viewType);
        int layoutId = itemViewDelegate.getItemViewLayoutId();
        ViewHolder holder = ViewHolder.createViewHolder(mContext, parent, layoutId);
        onViewHolderCreated(holder, holder.getConvertView());
        setListener(parent, holder, viewType);
        return holder;
    }

   public void convert(ViewHolder holder, T t) {
        mItemViewDelegateManager.convert(holder, t, holder.getAdapterPosition());
    }
}
```

##### 从上面看出，可以看到是通过`ItemViewDelegateManager`，来处理相关内容，再分析一下`ItemViewDelegateManager`

```java
public class ItemViewDelegateManager<T> {
    private SparseArrayCompat<ItemViewDelegate<T>> delegates = new SparseArrayCompat<>();

    public int getItemViewDelegateCount() {
        return delegates.size();
    }

    public void addDelegate(ItemViewDelegate<T> delegate) {
       	...
    }

    public ItemViewDelegateManager<T> removeDelegate(int itemType) {
	      ...
    }

    public int getItemViewType(T item, int position) {
        int delegatesCount = delegates.size();
        for (int i = delegatesCount - 1; i >= 0; i--) {
            ItemViewDelegate<T> delegate = delegates.valueAt(i);
            if (delegate.isForViewType(item, position)) {
                return delegates.keyAt(i);
            }
        }
        throw new IllegalArgumentException(
                "No ItemViewDelegate added that matches position=" + position + " in data source");
    }

    public void convert(ViewHolder holder, T item, int position) {
        int delegatesCount = delegates.size();
        for (int i = delegatesCount - 1; i >= 0; i--) {
            ItemViewDelegate<T> delegate = delegates.valueAt(i);

            if (delegate.isForViewType(item, position)) {
                delegate.convert(holder, item, position);
                return;
            }
        }
        throw new IllegalArgumentException(
                "No ItemViewDelegateManager added that matches position=" + position + " in data source");
    }
}

```



##### 看到这儿，基本就能看出它的基本实现原理，它是对系统方式的一种封装。它解决一些问题，

- 对itemview的具体实现做了封装处理
- 不需要静态去创建多个ViewHolder创建和缓存itemView，
- 多类型时减少if else判断。

##### 但它还暴露了一些问题：

- 增加一个itemType，需要修改adapter，使其支持新添加的类型。

- 类型处理比较麻烦

- 性能问题？

  

#### 针对以上的实现，开发一种方案，需要要考虑几个问题：

- 添加新类型时，只需要去实现ItemView的具体逻辑，不用操心添加类型支持等操作。

- ViewHolder不应该去针对每个类型做静态的创建，而是动态的创建。

- 获取itemType类型和查找绑定数据View的方法，应该减少逻辑操作。
- ItemView的数据绑定和展示要独立处理。



代码如下，提供一个抽象基类`MultiBaseAdapter`，根据系统提供的方法，要实现多类型，要修改以下方法。

##### 第一个要修改的方法。

```java
@Override
public int getItemViewType(int position) {
    if (mDataList != null && mDataList.get(position) != null) {
        return mDataList.get(position).getViewType();
    } else { // 托底操作
        return -1;
    }
}
```

##### 第二个方法，绑定数据的时候，把绑定操作，放在viewHolder（VH）内，

```java
@Override
public void onBindViewHolder(@NonNull VH viewHolder, int position) {
    T news = mDataList.get(position);
    viewHolder.bindData(position, news, mItemEvent);
}
```

查看VH类，在这里能IItemView进行数据的绑定。

```java
public class VH extends RecyclerView.ViewHolder {
    private final IItemVew itemVew;

    public VH(IItemVew itemView) {
        super((View) itemView.getItemView());
        this.itemVew = itemView;
    }

    <T extends IItemData> void bindData(int position, @NonNull T data, @Nullable IItemEvent itemEvent) {
        try {
            itemVew.setData(position, data, itemEvent);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

具体的itemView的写法。

```
public class ItemView0 extends BaseItemView implements IItemVew {

    public ItemView0(Context context) {
        super(context);
    }

    @Override
    protected int getContentView() {
        return R.layout.item_layout_0;
    }

    @SuppressLint("SetTextI18n")
    @Override
    public void setData(int position, IItemData data, IItemEvent itemEvent) {
        TextView textView = findViewById(R.id.textView);
        textView.setText("类型: " + data.getViewType());
        setBackgroundColor(Color.parseColor(ColorTemplate.COLOR_0));
        setOnClickListener(v -> itemEvent.onItemClick(ItemView0.this, position, data));
    }
}
```



##### 第三个方法onCreateViewHolder交给通用adapter类来创建，MultiBaseAdapter不再实现。

```java
@NonNull
@Override
public abstract VH onCreateViewHolder(@NonNull ViewGroup parent, int viewType);
```

##### 通用Adpter的写法如下

```java
private static class SimpleAdapter extends MultiBaseAdapter<ItemData> {

    public SimpleAdapter(IItemEvent itemEvent) {
        super(itemEvent);
    }
    @NonNull
    @Override
    public VH onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        IItemVew view = ItemViewFactory.getViewProxy(parent.getContext(), viewType);
        return new VH(view);
    }
}
```

这个类中会出现一个ItemViewFactory的类，这个类用来真正itemView的创建。它通过调用类的构造方法，来创建实例，另外每增加一个itemType，需要在`getViewClass`方法添加一条if语句,来返回Item的class。所以增加一种ItemType真正操作，是放在这里。

```
public class ItemViewFactory {
    @SuppressWarnings("unchecked")
    public static <T extends IItemVew> T getViewProxy(Context context, int viewType) {

        try {
            Class clazz = getViewClass(viewType);
            final Object o = clazz.getConstructor(Context.class).newInstance(context);
            return (T) Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(), new InvocationHandler() {
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    return method.invoke(o, args);
                }
            });
        } catch (Exception e) {
            return (T) new ItemViewDefault(context);
        }
    }
    private static Class<? extends IItemVew> getViewClass(int viewType) {
        if(viewType ==2){
            return ItemView2.class;
        }
        if(viewType ==0){
            return ItemView0.class;
        }
        if(viewType ==1){
            return ItemView1.class;
        }
        return ItemViewDefault.class;
    }
}
```

BaseItemView暂时不用管，只是处理一些通用item要处理的东西，IItemView是一个接口，用定统一描述一个itemView。



通过上面代码分析，要新加一种itemType类型的View，只需要修改ItemViewFactory方法，并且新建一个itemView的类，在里面实现具体逻辑就可以了。

对比看一下，可以基本上满足需求。

但上面立的flag中，【添加一种类型时，只需要关注所添加类型的View】就没有实现，因为我们每添加一种类型都需要修改`ItemViewFactory`来使其支持，这个就比较麻烦。



要增加一个itemType的新类型，需要修改ItemViewFactory这个简单工厂类，那能不能让这个类的创建自动化呢？





















。。。























#### 通过注解

简单了解一下注解（注意不是注释），可能有的iOS同学了解不多。

官网解释如下：<u>Java 注解用于为 Java 代码提供元数据。作为元数据，注解不直接影响你的代码执行，但也有一些类型的注解实际上可以用于这一目的。</u>

其实看了上面的话，还是不懂。我个人对注解的了解，分成两个词儿，**标注和解释**。要想使用注解，必须先创建一个注解，然后把注解标注的某个地方，然后在适当的时候，去解释这个注解标识的地方。

针对上面的问题，先创建两个注解，是针对类，并且是编译期的注解， 一个是Item注解，一个是Path注解。

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface Item {
    int [] type();
}
```

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.CLASS)
public @interface Path {
    String path();
}
```



对Model进行一个Path注解标记，这个字符串值，就是要创建ItemViewFactory的目录，可以自行修改，一般以包名为目录即可。此步骤只需要创建这个model的时候标记一次就可以了。

```
@Path(path = "com.yuan.recyclerviewmultyitemdemo")
public class ItemData implements IItemData {
```

对ItemView进行Item注解标记，这个标记用于标注此ItemView是用显示哪个itemType类型的数据，所以新加一个类时，只需要在类上添加此注解，标明支持的类型就可以了，这样就完成上面所说的【添加一种类型时，只需要关注所添加类型的View】。

```
@Item(type = 0)
public class ItemView0 extends BaseItemView implements IItemVew {
```

添加完注解后，只要代码编译，就可以生成ItemViewFactory类，并自动扫描哪些IItemView拥有Item注解，将这个itemView添加到ItemViewFactory的类型支持中。

它是通过注解处理器，来进行代码的处理，并不是在运行的时候才处理，所以并不会影响运行效果。

#### 思路

运用注解的方式来处理问题，在一些框架上使用并不在少数，Android第三方框架`EventBus`，`ButterKnife`，`Dagger`等。在后端更不在少数，比如`springboot`框架，`mybaties`(数据库处理框架)，`lombok`框架（javaBean处理)等。

简单看一个lombok框架

```
@NoArgsConstructor
@AllArgsConstructor
@ToString
@Builder
@NonNull
@Data
@Getter
@Setter
public class Data {
    private String value;
}
```







#### 思考？

组件化中各模块的通信问题。

设计组件化目标之一，就是各模块之间要解耦，但某些模块会有互相调用的问题，假如有两个模块，登录和分享，比如分享时需要调用登录模块的判断是否登录态等？

##### 通用处理方法

- 登录模块提供一个独立接口模块，并在模块中声明判断是否登录的方法。
- 登录模块实现独立接口模块中是否登录的方法。
- 打包或上传登录的独立接口模块
- 分享模块依赖登录的独立接口模块。
- 调用接口中的方法来判断是否登录。

这样做的问题在于，每增加一个实现，需要提供方首先在接口中声明，然后实现，再提供独立的包给使用者使用。

##### 是否也可以通过注解来实现？





#### 小彩蛋



#### 





