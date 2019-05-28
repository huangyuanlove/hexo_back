---
title: JetPack中的LiveData、ViewModel、LifeCycle
tags: [Android]
date: 2019-05-28 22:50:49
keywords: Android,Jetpack,LiveData,LifeCycle,ViewModel \
---
2018年谷歌I/O 发布了一系列辅助android开发者的实用工具，合称Jetpack，以帮助开发者构建出色的 Android 应用。
这次发布的 Android Jetpack 组件覆盖以下 4 个方面：Architecture、Foundation、Behavior 以及 UI。
包括我们在本次 Android P Beta 中带来的 Slices等新功能也包含在其中。
此外，Android Jetpack 完美兼容 Kotlin 语言，利用 Android KTX 可大幅节省代码量。
作为下一代的 Android 组件，Android Jetpack 通过提供现代化应用架构以及提供强健的向后兼容能力等方式，让开发者能够快速、轻松地创造拥有卓越性能的高质量应用。
<!--more-->

#### lifecycle  


原文：https://developer.android.google.cn/topic/libraries/architecture/lifecycle
说白了，就是一个接口回调，可以使用注解的方式来相应声明周期的回调，并且已经帮我们处理好了各种各样的意外状况(比如我们在onStart中做了比较多的操作，用户点了home键，导致onStop在OnStart完成之前被调用了)

Lifecycle是一个包含有关组件生命周期状态的信息(如Activity或Fragment)的类，允许其他对象观察此状态
Lifecycle使用两个枚举类`Event`和`State`来追踪组件的生命周期
![LifeCycler](/image/Android/jetpack/lifecycle-states.png)
代码如下
定义一个Observer实现LifecycleObserver接口
``` java 
public class MyLifecycleObserver implements LifecycleObserver {
    private String tag;
    public MyLifecycleObserver(String tag){
        this.tag = tag;
    }
    @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
    public void onCreate(){
        Log.e(tag,"ON_CREATE");
    }
    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    public void onStart(){
        Log.e(tag,"ON_START");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void onResume(){
        Log.e(tag,"ON_RESUME");
    }
    
    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void onPause(){
        Log.e(tag,"ON_PAUSE");
    }
    
    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    public void onStop(){
        Log.e(tag,"ON_STOP");
    }
    
    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    public void onDestroy(){
        Log.e(tag,"ON_DESTROY");
    }
}
```
在Activity中
``` java
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_lifecycle_observer);
        getLifecycle().addObserver(new MyLifecycleObserver("Activity"));
    }
```
这里要求Activity继承自实现了`LifecycleOwner`接口的类，比如`AppCompatActivity`等。
如果因为某些原因没有办法继承这种Activity，我们可以自己实现`LifecycleOwner`接口
``` java
public class WithoutLifeCycleActivity extends Activity implements LifecycleOwner {

    private LifecycleRegistry lifecycleRegistry;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_without_life_cycle);
        lifecycleRegistry = new LifecycleRegistry(this);
        lifecycleRegistry.markState(Lifecycle.State.CREATED);
        getLifecycle().addObserver(new MyLifecycleObserver("lifecycleRegistry"));
    }
    @Override
    protected void onStart() {
        super.onStart();
        lifecycleRegistry.markState(Lifecycle.State.STARTED);
    }
    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return lifecycleRegistry;
    }
}

```

#### LiveData
原文：https://developer.android.google.cn/topic/libraries/architecture/livedata
LiveData和RxJava很像，但是LiveData可以和LifeCycle绑定，来防止内存泄漏。因为LiveData仅通知处于活动状态的观察者有关更新的信息，
注册观查LiveData对象的非活动观察者不会收到有关更改的通知(当然我们也可以声明任何状态下都能手打通知)。
如果Observer类表示的观察者生命周期处于STARTED或RESUMED状态，则LiveData会将其视为处于活动状态。 
``` java
public class LiveDataActivity extends AppCompatActivity {

    private MutableLiveData<String> liveData;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_live_data);
        liveData = new MutableLiveData<>();
        liveData.observe(this, new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.e("observe",s);
            }
        });
        liveData.observeForever(new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.e("observeForever",s);
            }
        });
        liveData.setValue("onCreate");
    }
    
    @Override
    protected void onStart() {
        super.onStart();
        liveData.setValue("onStart");
    }
    
    @Override
    protected void onResume() {
        super.onResume();
        liveData.setValue("onResume");
    }
    
    @Override
    protected void onStop() {
        super.onStop();
        liveData.setValue("onStop");
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        liveData.setValue("onDestroy");
    }
}

```

我们打开界面,按下home键，然后返回到界面，然后按返回键，可以发现如下log：
打开界面
> E/observeForever: onCreate
  E/observeForever: onStart
  E/observe: onStart
  E/observe: onResume
  E/observeForever: onResume

按下home键
> E/observeForever: onStop

返回界面
> E/observeForever: onStart
  E/observe: onStart
  E/observe: onResume
  E/observeForever: onResume

按下返回键
> E/observeForever: onStop
  E/observeForever: onDestroy

如果我们将LiveData和LifeCycle绑定后，在界面处于非活动状态是收不到通知的。
LiveData经常和Room及ViewModel一块使用

#### ViewModel
原文：https://developer.android.google.cn/topic/libraries/architecture/viewmodel
ViewModel是以关联生命周期的方式来存储和管理UI相关的数据的类，即使configuration发生改变（比如旋转屏幕），数据仍然可以存在不会销毁。并且我们还可以通过ViewModelProviders在不同的Fragment之间共享数据。
实现ViewModel，这里省略掉了getter和setter方法。其实我们可以把每个属性都用`MutableLiveData`包装一下，这样当属性发生变化的时候，我们就可以立刻知道。

``` java

public class UserViewModel extends AndroidViewModel {
    public UserViewModel(@NonNull Application application) {
        super(application);
    }
    private String name;
    private int age;
    private String sex;
    private MutableLiveData<Integer> valueChanged = new MutableLiveData<>();

}
```

在Activity中
``` java

public class ViewModelActivity extends AppCompatActivity {

    ActivityViewModelBinding binding;
    private UserViewModel userViewModel;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding = DataBindingUtil.setContentView(this,R.layout.activity_view_model);
        userViewModel = ViewModelProviders.of(this).get(UserViewModel.class);
        userViewModel.getValueChanged().observeForever(new Observer<Integer>() {
            @Override
            public void onChanged(Integer integer) {
                binding.setUser(userViewModel);
            }
        });


        binding.save.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                userViewModel.setSex(binding.sex.getText().toString());
                userViewModel.setName(binding.name.getText().toString());
                userViewModel.setAge(Integer.valueOf(binding.age.getText().toString()));
                userViewModel.getValueChanged().postValue(1);
            }
        });
    
        binding.reset.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                userViewModel.setAge(11);
                userViewModel.setName("aa");
                userViewModel.setSex("M");
                userViewModel.getValueChanged().postValue(1);
                binding.setUser(userViewModel);
            }
        });
        binding.setUser(userViewModel);
    
        ArrayList<Fragment> pages = new ArrayList<>();
        pages.add(new ViewModelFragmentA());
        pages.add(new ViewModelFragmentB());
        PagerAdapter adapter=new ViewAdapter(getSupportFragmentManager(), pages);
       binding.viewPager.setAdapter(adapter);
    
    }
    
    class ViewAdapter extends FragmentPagerAdapter {
        private ArrayList<Fragment> list;
        public ViewAdapter(FragmentManager fm,ArrayList<Fragment> list) {
            super(fm);
            this.list=list;
        }
    
        @Override
        public Fragment getItem(int arg0) {
            return list.get(arg0);
        }
    
        @Override
        public int getCount() {
            return list.size();
        }
    }
}
```
FragmentA和FragmentB是一样的代码
``` java
public class ViewModelFragmentA extends Fragment implements View.OnClickListener {

    private EditText name;
    private EditText age;
    private EditText sex;
    
    private UserViewModel user;
    
    public ViewModelFragmentA() {
        // Required empty public constructor
    }
    
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    
        user = ViewModelProviders.of(getActivity()).get(UserViewModel.class);
    }
    
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        // Inflate the layout for this fragment
        View view = inflater.inflate(R.layout.fragment_view_model, container, false);
        name = view.findViewById(R.id.name);
        age = view.findViewById(R.id.age);
        sex = view.findViewById(R.id.sex);
    
        view.findViewById(R.id.save).setOnClickListener(this);
        view.findViewById(R.id.reset).setOnClickListener(this);
        view.findViewById(R.id.show).setOnClickListener(this);
        user.getValueChanged().observeForever(new Observer<Integer>() {
            @Override
            public void onChanged(Integer integer) {
                show();
            }
        });
    
        show();
        return view;
    }


    private void show() {
        sex.setText(user.getSex());
        age.setText(String.valueOf(user.getAge()));
        name.setText(user.getName());
    }
    
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.save:
                user.setSex(sex.getText().toString());
                user.setName(name.getText().toString());
                user.setAge(Integer.valueOf(age.getText().toString()));
                user.getValueChanged().postValue(1);
                break;
            case R.id.reset:
                user.setAge(11);
                user.setName("aa");
                user.setSex("M");
                user.getValueChanged().postValue(1);
                break;
            case R.id.show:
                show();
                break;
        }
    
    }
}
```
Activity中显示名字、年龄、性别。两个按钮save和reset。 在Fragment中多一个show按钮。
我们获取ViewModel都是通过`ViewModelProviders.of(getActivity()).get(UserViewModel.class)`来获取，得到是同一个对象。
当我们在任意一个地方修改数据并保存之后，会通过`valueChanged`属性来通知观察者来刷新界面。前面也提到过，我们可以把每个属性都用`MutableLiveData`
包装一下，观察者就可以在每个属性改变的时候得到通知了。

----
以上
