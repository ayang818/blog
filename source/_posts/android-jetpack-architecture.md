---
title: 使用Jetpack的Architecture工具降低Android开发中的耦合度
date: 2020-06-06 23:17:21
tags: [android, jetpack, 设计模式]
---

> 简单项目示例地址——https://github.com/ayang818/AndroidRookie
> 写这篇文章并不是因为要转android了（从大一上的时候光速学了三天，糊弄了一下android课的作业后就再没写过了QAQ），而是最近看到一篇关于在android使用MVC，MVP，MVVM的架构方法的[文章](https://www.tianmaying.com/tutorial/AndroidMVC)，感觉写的很好，就想来自己试一试android的几种解耦方法，触类旁通嘛~。此处也花了几个小时顺便快速学了一下android和其算是趋势的jetpack，故作此博客。

在编写Android代码的时候很容易在一个activity中就编写出耦合度特别高的代码，导致项目不利于维护。下面是几种常见的解决方案解决方案：
<!-- more -->
**1. UIData与activity耦合**

![UIData与activity耦合](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200606144343523.png)**2. 使用ViewModel将Model和Activity解耦**

![使用ViewModel](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200606144455182.png)

**3. 使用观察者模式（LiveData）主动刷新页面**

![使用MutableLiveData做事件监听](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200606144641412.png)

**4. 使用DataBinding进一步解决Controller和View之间的耦合**

![使用DataBinding解决C-V耦合](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200606144739564.png)

下面我们就会使用ViewModel，LiveData，DataBinding来实现页面的解耦。

### 使用ViewModel解除数据和界面之间的代码耦合

![](https://pic-serve.oss-cn-beijing.aliyuncs.com/blog/image-20200606140909659.png)

```java
myViewModel = new ViewModelProvider(this).get(MyViewModel.class);
```

ViewModel将UI中的数据抽离出来，统一维护。这样不会产生由于activity的摧毁（例如旋转屏幕）导致数据丢失的问题。

### 使用ViewModel + LiveData实现数据自动刷新代码示例

```java
public class ViewModelWithLifeData extends ViewModel {
    private MutableLiveData<Integer> likedNumber;

    public ViewModelWithLifeData() {
        likedNumber = new MutableLiveData<>();
        likedNumber.setValue(0);
    }

    public void addLikedNumber(int num) {
        likedNumber.setValue(likedNumber.getValue() + num);
    }

    public MutableLiveData<Integer> getLikedNumber() {
        return likedNumber;
    }
}
```

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    textView = findViewById(R.id.textView);
    likeButton = findViewById(R.id.imageButton);
    dislikeButton = findViewById(R.id.imageButton2);

    viewModel = new ViewModelProvider(this).get(ViewModelWithLifeData.class);
    // 观察者模式，设定回调
    viewModel.getLikedNumber().observe(this, (likedNumber) -> {
        textView.setText(String.valueOf(likedNumber));
    });

    likeButton.setOnClickListener((v) -> {
        viewModel.addLikedNumber(1);
    });

    dislikeButton.setOnClickListener((v) -> {
        viewModel.addLikedNumber(-1);
    });
}
```

### 使用DataBinding来完全解耦Model-View-Controller

首先在模块的build.gradle脚本的android.defaultConfig中加入，表示开启数据绑定模式

```groovy
dataBinding {
	enabled true
}
```

然后创建ViewModel

```java
public class ViewModelWithLifeData extends ViewModel {
    private MutableLiveData<Integer> likedNumber;

    public ViewModelWithLifeData() {
        likedNumber = new MutableLiveData<>();
        likedNumber.setValue(0);
    }

    public void addLikedNumber(int num) {
        likedNumber.setValue(likedNumber.getValue() + num);
    }

    public MutableLiveData<Integer> getLikedNumber() {
        return likedNumber;
    }
}
```

在activity中只需要编写Controller相关代码

```java
public class MainActivity extends AppCompatActivity {

    ViewModelWithLifeData viewModel;
    ActivityMainBinding bd;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        viewModel = new ViewModelProvider(this).get(ViewModelWithLifeData.class);
        // 绑定activity和其对应的xml
        bd = DataBindingUtil.setContentView(this, R.layout.activity_main);
        // DataBinding绑定对应ViewModel
        bd.setData(viewModel);
        // 设置生命周期的拥有者, 设置数据的监听
        bd.setLifecycleOwner(this);
    }
}
```

然后将activity的xml修改成binding模式，主要区别在加入了一对layout标签和一

对data标签，接着就可以通过xml来解决少量的视图逻辑，此处类似于数据的回绑

```xml
<data>
    <variable
        name="data"
        type="com.ayang818.livedata.ViewModelWithLifeData" />
</data>
......
<TextView
	...
	android:text="@{String.valueOf(data.likedNumber)}"
	/>
<ImageButton
	...
	android:onClick="@{() -> data.addLikedNumber(1)}"
	/>

```

这样我们用jetpack提供的architecture工具来完成M-V-C之间的解耦，项目也会更易于维护。