### Android View的绘制流程

设置参数

这里主要分析一下从xml文件到View对象的转换，我们有两种方式在java文件中引入xml：

- 通过LayoutInflater.inflate()方法。

##### setContentView(int layoutResID)方法分析

首先从我们最常用的AppCompatActivity中分析，可以看到调用了getDelegate()得到AppCompatDelegate(这个类随后分析)，然后重写于Activity方法中的setContentView方法。

```java
    @Override
    public void setContentView(@LayoutRes int layoutResID) {
        getDelegate().setContentView(layoutResID);
    }
```

我们点到Activity类中，可以看到有三个重载的setContentView方法

```java
public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
}
public void setContentView(View view) {
        getWindow().setContentView(view);
        initWindowDecorActionBar();
}
public void setContentView(View view, ViewGroup.LayoutParams params) {
        getWindow().setContentView(view, params);
        initWindowDecorActionBar();
 }
```

三个方法都调用了getWindow()方法，得到一个Windowd对象。

```java
 public Window getWindow() {
        return mWindow;
 }
```

Window对象是一个抽象类,这里简单介绍一下Window类，Window类作为一个抽象类里面封装了窗体样式和行为策略。它的实例作为顶层view被加到Window manager中，它提供了标准的UI策略例如：背景，标题栏，默认的按键事件处理(Windowd的interface Callback)等，它只有一个实现类PhoneWindow。

```JAVA
public abstract class Window
```

我们找一下在Activity类中mWindow什么时候赋值，可以看到在attach(…)方法中赋值，并将其回调方法给Activity，这里就可以知道每个Activity都有一个自己的PhoneWindow。

```Java
mWindow = new PhoneWindow(this, window, activityConfigCallback);
mWindow.setCallback(this);
```

继续setContentView(@LayoutRes int layoutResID)方法,我们找到PhoneWindow的setContentView方法，

```java
@Override
 public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }
		//当我们没有设置转场动画的时候
        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```

- 当我们设置转场动画的时候：
- 可以看到当我们没有设置转场动画的时候调用了mLayoutInflater.inflate(layoutResID, mContentParent)，mLayoutInflater在PhoneWindow的初始化方法中进行了初始化。

```java
 public PhoneWindow(Context context) {
        super(context);
        mLayoutInflater = LayoutInflater.from(context);
 }
```

我们看一下LayoutInflate的的创建方式，首先LayoutInflater是一个抽象类，发现有个sataic方法可以返回LayoutInflate对象。

```java

    /**
     * Obtains the LayoutInflater from the given context.
     */
    public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater =
                (LayoutInflater) 			 context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
```

我们在PhoneWindow的构造方法中就是通过这种方式创建的，在Activity中的getLayoutInfalte()方法也是获取到的PhoneWindow里面的LayoutInflater。

```java
public LayoutInflater getLayoutInflater() {
    return getWindow().getLayoutInflater();
}    
```

然后最终都还是调用了

```java
LayoutInflater LayoutInflater =
 (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```

总结一下我们可以通过三种方式获取到LayoutInflater对象，但最终还是调用的第三种方法：

- 在mActivity中通过getLayoutInflater()
- 通过LayoutInflater.from(context)方法
- 通过context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);

#####LayoutInfalte.inflate

我们继续分析Activity的setContentView()方法中的mLayoutInflater.inflate(layoutResID, mContentParent)方法，我们打开LayoutInfalte类中，可以看到四个重载的infalte()方法，我们这里分析两个需要传入layoutResID的方法

```java
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
    
```

可以看到第二个方法是内容，首先通过Resources把传进来的layoutResID转化为XmlResourceParser（根据对应的layoutResID生成相应的XmlResourceParser），然后调用inflate(parser, root, attachToRoot)方法，这里先总结一下这两个方法,因为最终都是调用的inflate(parser, root, attachToRoot)，我们反过来看一下。首先，parser是通过layoutResID创建的不能为空，root是可为空的，attachToRoot在第一种方法中是和root有关系的(当root不是null，为true,当root为null,则是false)。

接着我们来分析inflate(parser, root, attachToRoot)方法,这个方法最后会返回一个View。

```Java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot){
    synchronized (mConstructorArgs) {
       Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

       final Context inflaterContext = mContext;
       final AttributeSet attrs = Xml.asAttributeSet(parser);
       Context lastContext = (Context) mConstructorArgs[0];
       mConstructorArgs[0] = inflaterContext;
       //1.这里把root赋值给result，可以看到这个方法的返回值也是result.
       View result = root;

        try {
             // Look for the root node.
             int type;
             while ((type = parser.next()) != XmlPullParser.START_TAG &&
                type != XmlPullParser.END_DOCUMENT) {
                    // Empty
              }

              if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();

                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }
				
                if (TAG_MERGE.equals(name)) {
                  //2.如果根标签是merge,root不能为空，attachToRoot必须是true
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
					//3.在后面分析这个方法,如果根标签是merge的情况
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // 4.根据root和parser创建temp，点进去看实现，root可能为null.
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
					//5.创建LayoutParams
                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                      //6.当root不为空，attachToRoot为false时，temp根据设置了LayoutParams
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
                    //7.这里，如果root不为null,并且attachToRoot为true时，这里是把子View添加进root，并且设置了LayoutParams

                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                 	//8.当root为null的时候，result被设置为了刚才xml创建的布局。
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                final InflateException ie = new InflateException(e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } catch (Exception e) {
                final InflateException ie = new InflateException(parser.getPositionDescription()
                        + ": " + e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;

                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            return result;
        }
    }
```

这个方法可以看到分为三种情况

- root为null，也就是ViewGroup root为null，创建的是没有最外层layoutParams的View。
- root不为null,attachToRoot为true,是创建出有layoutParams的View，并添加到ViewGroup root中。
- root不为null,attachToRoot为false,是创建出有layoutParams的View，并返回。

上面的第3步会调用到这个方法rInflate(parser, root, inflaterContext, attrs, false);，

```Java
	private static final String TAG_MERGE = "merge";
    private static final String TAG_INCLUDE = "include";
    private static final String TAG_1995 = "blink";
    private static final String TAG_REQUEST_FOCUS = "requestFocus";
    private static final String TAG_TAG = "tag";
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;
        boolean pendingRequestFocus = false;
		//1.这里对xml文件进行遍历
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
			//2.当标签为requestFocus时
            if (TAG_REQUEST_FOCUS.equals(name)) {
                pendingRequestFocus = true;
                consumeChildElements(parser);
              //3.当标签为tag时
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
               //4.当标签为include时，include标签不能是根标签
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
              	//解析incliude标签
                parseInclude(parser, context, parent, attrs);
              //5.当标签为merge时，当标签为merge时标签必须是根标签
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else {
                //6.通过tag创建view，createViewFromTag(parent, name, context, attrs)创建
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
         		//7.把创建的view加到viewgroup中。
                viewGroup.addView(view, params);
            }
        }

        if (pendingRequestFocus) {
            parent.restoreDefaultFocus();
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
```

