title: LayoutInflater中inflate的使用方法
date: 2016-02-23 22:25:09
tags: [LayoutInflater]
categories: [Mobile,Android]
---
# 概述
本文记录 LayoutInflater 的 inflate 方法的三个参数的含义.

<!-- more -->

常常需要使用 LayoutInflater 中 inflate 方法。这个方法有两个重载的版本，一个是含有三个参数的，一个是两个参数的。

```java
//resource 代表了要被加载的布局文件的 ID，root 是待附加的父布局
public View inflate (int resource, ViewGroup root)
//前两个是一样的，最后一个 attachToRoot 代表是否加载到父布局的树形结构中
public View inflate (int resource, ViewGroup root, boolean attachToRoot) 
```

我们使用最频繁的地方应该就是在 adapter 的 getView 方法中，将 root 值设为 null，或者设一个 parent，将 attachToRoot 设为 false。一般没有特殊的需求的话，这样就可以了。

```java
 @Override
    public View getView(int position, View convertView, ViewGroup parent) {
        View root;
        ViewHolder viewHolder;
        if (convertView == null) {
            viewHolder = new ViewHolder();
            //加载布局
            root = LayoutInflater.from(mContext).inflate(R.layout.listitem_campaign, parent, false);
            //加载布局，上面是含有第三个参数的版本，下面是含有两个参数的版本。和上面的效果是一样的；
            //root = LayoutInflater.from(mContext).inflate(R.layout.listitem_campaign,null);
            viewHolder.time = (TextView) root.findViewById(R.id.campaignItemTimeTextView);
            viewHolder.imageView = (ImageView) root.findViewById(R.id.campaignItemImageView);
            viewHolder.title = (TextView) root.findViewById(R.id.campaignItemTitleTextView);
            viewHolder.desc = (TextView) root.findViewById(R.id.campaignItemDescTextView);
            viewHolder.redCircle = (ImageView) root.findViewById(R.id.redCircleImageView);
            root.setTag(viewHolder);
        } else {
            root = convertView;
            viewHolder = (ViewHolder) root.getTag();
        }
        Campaign campaign = data.get(position);
        viewHolder.time.setText(campaign.getTime());
        viewHolder.title.setText(campaign.getTitle());
        viewHolder.desc.setText(campaign.getDescription());
        viewHolder.time.setText(campaign.getTime());
        viewHolder.redCircle.setVisibility(campaign.isRead() ? View.INVISIBLE : View.VISIBLE);
        return root;
    }

```

当 parent 为 null 的时候，attachToRoot 就为 false。
```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }
```
再看下 inflate 方法的关键部分。
```java
// Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                } 
```
当 root 为空的时候，直接返回了 temp (temp是从xml布局文件中加载的view)，而当 root 不为空的时候，会将 root 作为父布局，根据 xml 解析布局文件中的节点，获取属性元素，重新生成 temp 的布局参数 params (此时假如 attachToRoot 为 false，则会将 temp 的布局参数设置成生成的布局参数 params),而后根据 temp 重新inflate temp 中的子 view，该设置 params 的设置 params。而后如果 parent 不为空，attachToRoot 为 true，就会将整个布局中的所有元素挨个添加到 parent 中。最后返回的是 parent。

那么当 parent 为空的时候，又是什么个情况呢？因为 parent 为空，也就是 xml 文件根布局没有 parent 作为参照，解析的时候，他的高无论节点上设置的是什么都会默认是 wrap_content，宽都会是 match_parent。


假设目前有个布局文件为
```xml
<?xml version="1.0" encoding="utf-8"?>
<TextView
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="25dp"
        android:layout_height="25dp"
        android:background="#ff0000"
        android:text="red"/>
```

这里我们指定了 textview 的宽高。这里的 textview 就一个布局文件的跟布局。看下下面的代码，adapter 的 item 的布局也是类似的，原理一样。

```java
public class LayoutInflaterActivity extends AppCompatActivity {

    private View view = null;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_layoutinflater_main);

        ViewGroup parent = (ViewGroup) findViewById(R.id.layoutInflaterContainer);
        // result: layout_height=wrap_content layout_width=match_parent
        view = LayoutInflater.from(this).inflate(R.layout.activity_layoutinflater_sub, null);
        parent.addView(view);

        // result: layout_height=100 layout_width=100
        view = LayoutInflater.from(this).inflate(R.layout.activity_layoutinflater_sub, null);
        parent.addView(view, 100, 100);

        // result: layout_height=25dp layout_width=25dp
        // view=textView due to attachRoot=false
        view = LayoutInflater.from(this).inflate(R.layout.activity_layoutinflater_sub, parent, false);
        parent.addView(view);

        // result: layout_height=25dp layout_width=25dp
        // parent.addView not necessary as this is already done by attachRoot=true
        // view=root due to parent supplied as hierarchy root and attachRoot=true
        view = LayoutInflater.from(this).inflate(R.layout.activity_layoutinflater_sub, parent, true);
    }
}

```

- 第一种方案，

```java 
        // result: layout_height=wrap_content layout_width=match_parent
        view = LayoutInflater.from(this).inflate(R.layout.activity_layoutinflater_sub, null);
        parent.addView(view);
```

效果就是加载的布局宽度为充满 match_parent ，高度为 wrap_content。


- 第二种方案

```java
        // result: layout_height=100 layout_width=100
        view = LayoutInflater.from(this).inflate(R.layout.activity_layoutinflater_sub, null);
        parent.addView(view, 100, 100);
```
效果就是加载的布局宽宽高均为 100 个像素。


- 第三种方案

```java 
        // result: layout_height=25dp layout_width=25dp
        // view=textView due to attachRoot=false
        view = LayoutInflater.from(this).inflate(R.layout.activity_layoutinflater_sub, parent, false);
        parent.addView(view);
```

效果就是加载的布局宽宽高均为布局中设置的 25dp。

- 第四种

```java 
        // result: layout_height=25dp layout_width=25dp
        // parent.addView not necessary as this is already done by attachRoot=true
        // view=root due to parent supplied as hierarchy root and attachRoot=true
        view = LayoutInflater.from(this).inflate(R.layout.activity_layoutinflater_sub, parent, true);
```
效果就是加载的布局宽宽高均为布局中设置的 25dp，并且添加到了当前的布局中。

## 总结

所以，要能在 getview 中，自定义宽高，办法有两个，一个是在最外层再套一层布局，然后 parent 设为 null。另一个是指定 parent，将 attachToRoot 设为 false。
