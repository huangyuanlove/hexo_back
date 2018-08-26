---
title: Coordinatorlayout
date: 2018-08-20 23:16:18
tags: [Android]
keywords: Coordinatorlayout
photos: 
  - /image/Android/Coordinatorlayout/Coordinatorlayout.gif

---
上图的动画其实挺简单的，如果你知道的话，就不要继续往下看了，那是在浪费时间。
<!--more-->
主要用的前几年推出的几个support包，可惜国内没有流行起来。

简单直接放代码：
gradle依赖：
``` gradle
  implementation 'com.android.support:appcompat-v7:28.0.0-rc01'
  implementation 'com.android.support.constraint:constraint-layout:1.1.2'
  implementation 'com.android.support:design:28.0.0-rc01'
```

布局文件：
``` xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.design.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:fitsSystemWindows="true"
    android:layout_height="match_parent">

    <android.support.design.widget.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="160dp"
        android:fitsSystemWindows="true"
        android:theme="@style/ThemeOverlay.AppCompat.ActionBar">

        <android.support.design.widget.CollapsingToolbarLayout
            android:id="@+id/collapsingToolbarLayout"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:fitsSystemWindows="true"
            app:collapsedTitleTextAppearance="@style/TextAppearance.AppCompat.Title"
            app:contentScrim="?attr/colorPrimaryDark"
            app:expandedTitleMarginStart="48dp"
            app:expandedTitleTextAppearance="@style/TextAppearance.AppCompat.Title"
            app:layout_scrollFlags="scroll|exitUntilCollapsed">

            <ImageView
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:fitsSystemWindows="true"
                android:scaleType="fitXY"
                android:src="@drawable/title_bg"
                app:layout_collapseMode="parallax"
                app:layout_collapseParallaxMultiplier="0.5" />

            <android.support.v7.widget.Toolbar
                android:id="@+id/toolbar"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize" />
        </android.support.design.widget.CollapsingToolbarLayout>

    </android.support.design.widget.AppBarLayout>

    <android.support.v7.widget.RecyclerView
        android:id="@+id/recyclerview"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior" />


</android.support.design.widget.CoordinatorLayout>
```

activity代码：
``` java

public class MainActivity extends AppCompatActivity {

    private List<String> data = new ArrayList<>();


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toolbar toolbar = findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        CollapsingToolbarLayout collapsingToolbarLayout = findViewById(R.id.collapsingToolbarLayout);
        collapsingToolbarLayout.setTitle("Test CoordinatorLayout");
        collapsingToolbarLayout.setContentScrimColor(Color.GRAY);
        collapsingToolbarLayout.setCollapsedTitleTextColor(ContextCompat.getColor(this,R.color.colorAccent));
        collapsingToolbarLayout.setExpandedTitleColor(Color.WHITE);
        for(int i = 0 ; i < 100 ;i ++){
            data.add("item -- >" + i);
        }
        RecyclerView recyclerView = findViewById(R.id.recyclerview);

        LinearLayoutManager linearLayoutManager = new LinearLayoutManager(MainActivity.this);
        linearLayoutManager.setOrientation(LinearLayoutManager.VERTICAL);
        recyclerView.setLayoutManager(linearLayoutManager);
        RecyclerViewAdapter adapter = new RecyclerViewAdapter();
        recyclerView.setAdapter(adapter);
    }


    class RecyclerViewAdapter extends RecyclerView.Adapter<RecyclerViewAdapter.ViewHolder>{


        @NonNull
        @Override
        public ViewHolder onCreateViewHolder(@NonNull ViewGroup viewGroup, int i) {
            View view = LayoutInflater.from(MainActivity.this).inflate(android.R.layout.simple_list_item_1,viewGroup,false);

            return new ViewHolder(view);
        }

        @Override
        public void onBindViewHolder(@NonNull ViewHolder viewHolder, int i) {

            viewHolder.textView.setText(data.get(i));
            viewHolder.textView.setTextColor(ContextCompat.getColor(MainActivity.this,R.color.colorAccent));


        }

        @Override
        public int getItemCount() {
            return data.size();
        }

        class ViewHolder extends RecyclerView.ViewHolder{

            TextView textView;

            public ViewHolder(@NonNull View itemView) {
                super(itemView);
                textView = itemView.findViewById(android.R.id.text1);
            }
        }
    }
}
```

不想多写什么了，可以自己去搜这些东西的用法。傲娇脸~_~

-------------------------------------------
以上