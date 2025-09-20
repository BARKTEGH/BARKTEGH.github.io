---
title: 安卓开发---天气
date: 2020-04-24 20:36:24
tags: 
- Android
categories:
- Android
copyright: ture
---


# 安卓开发---天气
----
### 技术选型
1. ViewPager  用来展示切换多个城市天气情况
2. SwipeRefreshLayout 下拉刷新天气
3. ScrollView 滑动展示天气详情
4. 参考第一行代码中CoolWeather

### 页面布局
天气页面的粗略布局如下：
主要为SwipeRefreshLayout嵌套ViewPager

    <FrameLayout>
		//展示背景图片
	    <ImageView/>
	    <LinearLayout>
	        <LinearLayout/>
			//下拉刷新
	        <SwipeRefreshLayout>	
				//展示天气页面
	            <ViewPager/>
	        </SwipeRefreshLayout>
	    </LinearLayout>
	</FrameLayout>

下面为viewpager的布局，ScrollView用来滑动展示单个城市的天气详情，LinearLayout里展示天气详情

	<FrameLayout>
		//纵向滑动显示天气详情
        <ScrollView>
            <LinearLayout>
                <include layout="@layout/now" />
                <include layout="@layout/forecast" />
                <include layout="@layout/lifestyle"/>
            </LinearLayout>
        </ScrollView>
	</FrameLayout>

### 遇到的问题

#### ViewPager和SwipeRefreshLayout发生滑动冲突

- 原因: SwipeRefreshLayout的onIntercept方法发现是在这里拦截了滑动事件，无法传递到viewpager里，假如无法像直线一样横向滑动出现了纵向偏差就会触发SwipeRefreshLayout

- 解决思路：下拉刷新只在纵向滑动触发，那么在SwipeRefreshLayout里只拦截纵向滑动


	    import android.content.Context;
		import android.util.AttributeSet;
		import android.view.MotionEvent;
		import android.view.ViewConfiguration;
		
		import androidx.annotation.NonNull;
		import androidx.swiperefreshlayout.widget.SwipeRefreshLayout;
		
		public class PocketSwipeRefreshLayout extends SwipeRefreshLayout {
		    private float mStartX = 0;
		    private float mStartY = 0;
		
		    //记录Viewpager是否被拖拉
		    private boolean mIsVpDrag;
		    private final int mTouchSlop;
		    public PocketSwipeRefreshLayout(@NonNull Context context) {
		        super(context);
		        mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
		    }
		
		    public PocketSwipeRefreshLayout(@NonNull Context context, AttributeSet attrs) {
		        super(context,attrs);
		        mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
		    }
		
		    @Override
		    public boolean onInterceptTouchEvent(MotionEvent ev) {
		        int action = ev.getAction();
		        switch (action){
		            case MotionEvent.ACTION_DOWN:
		                mStartX = ev.getX();
		                mStartY = ev.getY();
		                mIsVpDrag = false;
		                break;
		            case MotionEvent.ACTION_MOVE:
		                //如果viewpager正在拖拽，则不拦截viewpager事件
		                if (mIsVpDrag){
		                    return false;
		                }
		                float x = ev.getX();
		                float y = ev.getY();
		                float distanceX = Math.abs(x - mStartX);
		                float distanceY = Math.abs(y - mStartY);
		                //如果滑动x位移大于y，不拦截viewpager事件
		                if (distanceX > mTouchSlop && distanceX>distanceY){
		                    mIsVpDrag = true;
		                    return false;
		                }
		                break;
		            case MotionEvent.ACTION_UP:
		            case MotionEvent.ACTION_CANCEL:
		                mIsVpDrag = true;
		                break;
		            default:
		                break;
		        }
		        return super.onInterceptTouchEvent(ev);
		
		    }
		}


#### SwipeRefreshLayout与ScrollView发生下滑冲突
- 情况说明：在未采用ViewPager时，即只展示一个城市天气时，SwipeRefreshLayout与ScrollView并不会发生下滑冲突，但在采用ViewPager后会发生冲突，导致浏览到页面下部后却触发SwipeRefreshLayout无法向上滑动页面
- 原因：SwipeRefreshLayout拦截了纵向滑动事件，但为什么未采用viewpager是不会发生冲突还未明白
- 解决思路：当且仅当滑动到顶部时才需要触发下拉刷新，那么在纵坐标未在顶部时屏蔽SwipeRefreshLayout

		// 初始化各控件
        ScrollView weatherLayout = (ScrollView) view.findViewById(R.id.weather_layout);
        //解决SwipeRefreshLayout与ScrollView的下拉冲突
        weatherLayout.setOnScrollChangeListener(new View.OnScrollChangeListener() {
            @Override
            public void onScrollChange(View v, int scrollX, int scrollY, int oldScrollX, int oldScrollY) {
                if (swipeRefreshLayout != null){
                    swipeRefreshLayout.setEnabled(scrollY==0);
                }
            }
        });


#### ViewPager动态更新页面
- 情况说明：在下拉刷新时，需要去重新得到城市的天气情况并展示在UI中。
- 解决思路：PagerAdapter可以通过调用notifyDataSetChanged()方法实现数据集的刷新。
viewpager的刷新过程是这样的：在每次调用notifyDataSetChanged()时，都会激活getItemPosition(Object object)方法，该方法会遍历viewpager的所有item，为每个item返回一个状态值（POSITION_NONE/POSITION_UNCHANGED），如果是none，那么该item会被destroyItem(ViewGroup container, int position, Object object)方法remove掉，然后重新加载，如果是unchanged，就不会重新加载，默认是unchanged，所以如果我们不重写getItemPosition(Object object)，就无法看到刷新效果。
	

	那么为了只刷新单个页面，我们重写getItemPosition方法，并且为了确定需要刷新哪个页面，给每个view添加一个tag。


		viewPager = (ViewPager) findViewById(R.id.weather_pager);
        LayoutInflater layoutInflater = getLayoutInflater();
        for (int i=0;i<choosedCountyList.size(); i++){
            View view = layoutInflater.inflate(R.layout.weather_layout, null);
            view.setTag(i);
            String countyName = choosedCountyList.get(i);
            fillView(view, countyName,true);
            pageList.add(view);
        }
		adapter = new PagerAdapter() {
            @Override
            public int getCount() {
                return pageList.size();
            }

            @Override
            public int getItemPosition(@NonNull Object object) {
                //更新其中一个页面
                View view = (View) object;
                int currentPagerIdx = viewPager.getCurrentItem();
                if (currentPagerIdx == (Integer) view.getTag()) {
                    return POSITION_NONE;
                } else {
                    return POSITION_UNCHANGED;
                }
          
            }

            @Override
            public boolean isViewFromObject(@NonNull View view, @NonNull Object object) {
                return view==object;
            }

            @Override
            public void destroyItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
				container.removeView(pageList.get(position));
            }

            @NonNull
            @Override
            public Object instantiateItem(@NonNull ViewGroup container, int position) {
                container.addView(pageList.get(position));
                return pageList.get(position);
            }


        };


#### ViewPager页添加或减少城市
- 情况说明：
	- 当出现删除城市时，即删除viewPager中的view，如果直接pageList中remove页面后使用adapter.notifyDataSetChanged()来通通知视图改变，这会出现超出索引的问题。
	- 当出现增加城市，一般不会出现问题。
	- 当出现城市数量不变但是城市发生了改变，会出现显示城市错误的情况，仍然显示原来的城市天气情况。
- 原因： 
	- destroyItem方法中采用了位置索引的方式来删除页面，但是因为pageList中remove了页面，导致索引超出了。
	- 无问题
	- 因为viewPager会缓存当前页面的左右页面(3个页面)，这会导致仍然显示原来的城市
- 解决思路：
	-  destroyItem方法将object转为view后删除的方式来解决索引问题。
	-  清空pageList，重新添加城市，此时需要注意因为上面选择了只更新当前页面，但现在需要更新所有页面，设置标志位来辨别更新一个还是更新所有。

主要的代码变动如下：

	//改变适配器
	adapter = new PagerAdapter() {
	    @Override
	    public int getCount() {
	        return pageList.size();
	    }
	
	    @Override
	    public int getItemPosition(@NonNull Object object) {
	        if (updateStatu == UPDATE_ONE_PAGE){
	            //更新其中一个页面
	            View view = (View) object;
	            int currentPagerIdx = viewPager.getCurrentItem();
	            if (currentPagerIdx == (Integer) view.getTag()) {
	                return POSITION_NONE;
	            } else {
	                return POSITION_UNCHANGED;
	            }
	        } else if (updateStatu == UPDATE_ALL_PAGE){
	            //更新所有页面，目的为去除缓存
	            return POSITION_NONE;
           }
	        return POSITION_NONE;
	    }
	}


	//在城市列表方式变化是重新生成页面
	//当前变化
    if (!equalList(curList, choosedCountyList)){
        LayoutInflater layoutInflater = getLayoutInflater();
        pageList.clear();
        for (int i=0; i< curList.size();i++){
            //以前的没有，新增的城市，新增页面
            View view = layoutInflater.inflate(R.layout.weather_layout, null);
            view.setTag(i);
            String countyName = curList.get(i);
            fillView(view, countyName,false);
            pageList.add(i, view);
        }
        updateStatu = UPDATE_ALL_PAGE;
        adapter.notifyDataSetChanged();
    }
	//一定要在选择页面前更新城市列表，不然增加城市时onPageSelected出现超过索引的问题
    choosedCountyList = curList;
    viewPager.setCurrentItem(position);

	/**
     * 更新当前页天气
     */
    private void updatePageView(){
        updateStatu = UPDATE_ONE_PAGE;
        View view = pageList.get(position);
        String countyName = choosedCountyList.get(position);
        fillView(view, countyName, true);
        adapter.notifyDataSetChanged();
    }

#### Weather展示
- 情况：默认情况每次都会生成新的活动，但是天气展示页面只需要一个就够
- 解决思路: 在manifest文件中设置WeatherActivity为singleTask

		<activity android:name=".WeatherActivity" android:launchMode="singleTask"/>

#### 从城市添加页面跳转到Weather页面展示当前选中的城市
- 情况：从从城市添加页面跳转到Weather页面时默认显示之前Weather页面显示的城市情况
- 解决思路：在`prefs = PreferenceManager.getDefaultSharedPreferences
(this)`中保存当前选中的城市，随后在Weather活动中取出，根据城市名得到所在页面的索引，设置为当前页。`viewPager.setCurrentItem(position);`

	

### 代码展示

    <?xml version="1.0" encoding="utf-8"?>
	<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	    package="com.coolweather.android">
	
	    <!-- 这个权限用于进行网络定位 -->
	    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" /> <!-- 这个权限用于访问GPS定位 -->
	    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" /> <!-- 用于访问wifi网络信息，wifi信息会用于进行网络定位 -->
	    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" /> <!-- 获取运营商信息，用于支持提供运营商信息相关的接口 -->
	    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" /> <!-- 这个权限用于获取wifi的获取权限，wifi信息会用来进行网络定位 -->
	    <uses-permission android:name="android.permission.CHANGE_WIFI_STATE" /> <!-- 写入扩展存储，向扩展卡写入数据，用于写入离线定位数据 -->
	    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" /> <!-- 访问网络，网络定位需要上网 -->
	    <uses-permission android:name="android.permission.INTERNET" />
	    <uses-permission android:name="android.permission.READ_PHONE_STATE" />
	
	    <application
	        android:name="org.litepal.LitePalApplication"
	        android:allowBackup="true"
	        android:icon="@mipmap/ic_launcher"
	        android:label="@string/app_name"
	        android:roundIcon="@mipmap/ic_launcher_round"
	        android:supportsRtl="true"
	        android:theme="@style/AppTheme">
	        <service
	            android:name=".AutoUpdateService"
	            android:enabled="true"
	            android:exported="true"></service>
	
	        <meta-data
	            android:name="com.baidu.lbsapi.API_KEY"
	            android:value="xxxxxx" />
	
	        <activity android:name=".ChooseAreaActivity" />
	        <activity android:name=".CountyChoosedActivity" />
	
	        <activity android:name=".WeatherActivity"
	            android:launchMode="singleTask"/>
	
	        <activity android:name=".MainActivity">
	            <intent-filter>
	                <action android:name="android.intent.action.MAIN" />
	                <category android:name="android.intent.category.LAUNCHER" />
	            </intent-filter>
	        </activity>
	
	        <service
	            android:name="com.baidu.location.f"
	            android:enabled="true"
	            android:process=":remote" />
	    </application>
	
	</manifest>


MainActivity

    package com.coolweather.android;

	import androidx.annotation.NonNull;
	import androidx.appcompat.app.AppCompatActivity;
	import androidx.core.app.ActivityCompat;
	import androidx.core.content.ContextCompat;
	
	import android.Manifest;
	import android.content.Intent;
	import android.content.SharedPreferences;
	import android.content.pm.PackageManager;
	import android.icu.text.UnicodeSetSpanner;
	import android.os.Bundle;
	import android.preference.PreferenceManager;
	import android.widget.Toast;
	
	import com.baidu.location.BDLocation;
	import com.baidu.location.BDLocationListener;
	import com.baidu.location.LocationClient;
	import com.baidu.location.LocationClientOption;
	import com.baidu.mapapi.SDKInitializer;
	import com.coolweather.android.db.ChoosedCounty;
	import com.coolweather.android.util.BDLocationUtil;
	import com.coolweather.android.util.Utility;
	
	import java.util.ArrayList;
	import java.util.List;
	
	public class MainActivity extends AppCompatActivity {
	
	
	    private LocationClient locationClient;
	
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_main);
	
	        final SharedPreferences prefs = PreferenceManager.getDefaultSharedPreferences(this);
	        if (prefs.getString("weather",null)!=null) {
	            String countyName = prefs.getString("weatherCityName", "北京");
	            SharedPreferences.Editor edit = prefs.edit();
	            edit.putString("curCounty", countyName);
	            edit.apply();
	            Utility.saveChoosedCounty(countyName);
	            Intent intent = new Intent(this, WeatherActivity.class);
	            startActivity(intent);
	            finish();
	        } else {
	            locationClient = new LocationClient(getApplicationContext());
	            locationClient.registerLocationListener(new BDLocationListener() {
	                @Override
	                public void onReceiveLocation(final BDLocation location) {
	                    String district = location.getDistrict();
	                    if (district == null){
	                        runOnUiThread(new Runnable() {
	                            @Override
	                            public void run() {
	                                Toast.makeText(MainActivity.this,
	                                        "无法获取定位，重置定位为北京",
	                                        Toast.LENGTH_SHORT).show();
	                            }
	                        });
	                        district = "北京";
	                    }
	                    locationClient.stop();
	                    Utility.saveChoosedCounty(district);
	                    SharedPreferences.Editor edit = prefs.edit();
	                    edit.putString("curCounty", district);
	                    edit.apply();
	                    Intent intent = new Intent(MainActivity.this, WeatherActivity.class);
	                    startActivity(intent);
	                    finish();
	                }
	            });
	            SDKInitializer.initialize(getApplicationContext());
	            //申请权限
	            if (requestPermission()){
	                requestLocation();
	            }
	
	        }
	    }
	
	    private void requestLocation(){
	        LocationClientOption option = new LocationClientOption();
	        option.setIsNeedAddress(true);
	        locationClient.setLocOption(option);
	        locationClient.start();
	    }
	
	    /**
	     * 请求权限
	     */
	    private boolean requestPermission(){
	        List<String> permissionList = new ArrayList<>();
	        if (ContextCompat.checkSelfPermission(MainActivity.this, Manifest.
	                permission.ACCESS_FINE_LOCATION)!= PackageManager.PERMISSION_GRANTED) {
	            permissionList.add(Manifest.permission.ACCESS_FINE_LOCATION);
	        }
	        if (ContextCompat.checkSelfPermission(MainActivity.this, Manifest.
	                permission.WRITE_EXTERNAL_STORAGE)!= PackageManager.PERMISSION_GRANTED) {
	            permissionList.add(Manifest.permission.WRITE_EXTERNAL_STORAGE);
	        }
	        if (ContextCompat.checkSelfPermission(MainActivity.this,
	                Manifest.permission.ACCESS_COARSE_LOCATION)!= PackageManager.PERMISSION_GRANTED){
	            permissionList.add(Manifest.permission.ACCESS_COARSE_LOCATION);
	        }
	        if (ContextCompat.checkSelfPermission(MainActivity.this,
	                Manifest.permission.READ_PHONE_STATE) != PackageManager.PERMISSION_GRANTED){
	            permissionList.add(Manifest.permission.READ_PHONE_STATE);
	        }
	        if (!permissionList.isEmpty()) {
	            String [] permissions = permissionList.toArray(new String[permissionList.
	                    size()]);
	            ActivityCompat.requestPermissions(MainActivity.this, permissions, 1);
	            return false;
	        }
	        return true;
	    }
	
	    @Override
	    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
	        switch (requestCode) {
	            case 1:
	                if (grantResults.length > 0) {
	                    for (int result : grantResults) {
	                        if (result != PackageManager.PERMISSION_GRANTED) {
	                            Toast.makeText(this, "必须同意所有权限才能使用本程序",
	                                    Toast.LENGTH_SHORT).show();
	                            finish();
	                            return;
	                        }
	                    }
	                    requestLocation();
	                } else {
	                    Toast.makeText(this, "发生未知错误", Toast.LENGTH_SHORT).show();
	                }
	                break;
	            default:
	        }
	    }
	
	
	}


WeatherActivity

	package com.coolweather.android;

	import androidx.annotation.NonNull;
	import androidx.annotation.RequiresApi;
	import androidx.appcompat.app.AppCompatActivity;
	import androidx.swiperefreshlayout.widget.SwipeRefreshLayout;
	import androidx.viewpager.widget.PagerAdapter;
	import androidx.viewpager.widget.ViewPager;
	
	import android.annotation.SuppressLint;
	import android.content.Intent;
	import android.content.SharedPreferences;
	import android.graphics.Color;
	import android.os.Build;
	import android.os.Bundle;
	import android.preference.PreferenceManager;
	import android.util.ArraySet;
	import android.util.Log;
	import android.view.LayoutInflater;
	import android.view.View;
	import android.view.ViewGroup;
	import android.widget.Button;
	import android.widget.ImageView;
	import android.widget.LinearLayout;
	import android.widget.ScrollView;
	import android.widget.TextView;
	import android.widget.Toast;
	
	import com.bumptech.glide.Glide;
	import com.coolweather.android.db.ChoosedCounty;
	import com.coolweather.android.gson.Forecast;
	import com.coolweather.android.gson.Lifestyle;
	import com.coolweather.android.gson.Weather;
	import com.coolweather.android.util.HttpUtil;
	import com.coolweather.android.util.Utility;
	import com.coolweather.android.view.PocketSwipeRefreshLayout;
	
	
	import org.litepal.crud.DataSupport;
	
	import java.io.IOException;
	import java.util.ArrayList;
	import java.util.Arrays;
	import java.util.HashMap;
	import java.util.HashSet;
	import java.util.List;
	
	import okhttp3.Call;
	import okhttp3.Callback;
	import okhttp3.Response;
	
	@RequiresApi(api = Build.VERSION_CODES.M)
	public class WeatherActivity extends AppCompatActivity {
	
	    /**
	     * 采用singleTask模式，只存在一个weather活动
	     *
	     */
	    private static final String TAG = "WeatherActivity";
	
	    private PocketSwipeRefreshLayout swipeRefreshLayout;
	    private ImageView bingPIcImg;
	    private Button navButton;
	    private TextView titleCityText;
	    private LinearLayout dotLayout;
	
	    private SharedPreferences prefs;
	
	    private ViewPager viewPager;
	    private PagerAdapter adapter;
	    //保存每个页面view
	    private ArrayList<View> pageList = new ArrayList<>();
	    private List<ImageView> dotImageList = new ArrayList<>();
	    //当前页索引
	    private int position;
	    //当前显示的城市
	    private String curCountyName;
	
	    //所有选中的城市
	    private List<String> choosedCountyList = new ArrayList<>();
	    //城市对应的位置
	    private HashMap<String, Integer> countyIndex = new HashMap<>();
	
	    //更新一个页面
	    private int UPDATE_ONE_PAGE = 0;
	    //更新所有页面
	    private int UPDATE_ALL_PAGE = 1;
	    //更新标记，用来切换下拉刷新更新一个页面 和 减少页面更新所有页面来去除缓存
	    private int updateStatu = UPDATE_ONE_PAGE;
	
	
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_weather);
	        //实现状态栏配色统一
	        if (Build.VERSION.SDK_INT>=21){
	            View decorView = getWindow().getDecorView();
	            decorView.setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN|
	                    View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
	            getWindow().setStatusBarColor(Color.TRANSPARENT);
	        }
	        setContentView(R.layout.activity_weather);
	        //数据初始化-- 将传入的数据赋值到对应的位置
	        choosedCountyList = initChoosedCounties();
	        prefs = PreferenceManager.getDefaultSharedPreferences
	                (this);
	        curCountyName = prefs.getString("curCounty",choosedCountyList.get(0));
	        position = getPositionFromName(curCountyName, choosedCountyList);
	
	        //圆点来显示当前页面的位置
	        dotLayout = (LinearLayout) findViewById(R.id.dot_layout);
	        for (int i = 0; i < choosedCountyList.size(); i++) {
	            ImageView imageView = new ImageView(WeatherActivity.this);
	            imageView.setLayoutParams(new ViewGroup.LayoutParams(30, 30));
	            imageView.setPadding(20, 0, 20, 0);
	            if (i == position) {
	                // 默认选中第一张图片
	                imageView.setBackgroundResource(R.drawable.dot_focused);
	            } else {
	                imageView.setBackgroundResource(R.drawable.dot_normal);
	            }
	            dotImageList.add(imageView);
	            dotLayout.addView(imageView);
	        }
	
	
	        //下拉刷新
	        swipeRefreshLayout = (PocketSwipeRefreshLayout) findViewById(R.id.swipe_refresh);
	        swipeRefreshLayout.setColorSchemeResources(R.color.colorPrimary);
	        swipeRefreshLayout.setOnRefreshListener(new PocketSwipeRefreshLayout.OnRefreshListener() {
	            @Override
	            public void onRefresh() {
	                updatePageView();
	                Toast.makeText(WeatherActivity.this, "刷新",
	                        Toast.LENGTH_SHORT).show();
	                swipeRefreshLayout.setRefreshing(false);
	            }
	        });
	        //tupian
	        bingPIcImg = (ImageView) findViewById(R.id.bing_pic_img);
	        String bingPic = prefs.getString("bing_pic", null);
	        if (bingPic != null){
	            Glide.with(this).load(bingPic).into(bingPIcImg);
	        }else {
	            loadBingPIc();
	        }
	        //添加按钮
	        navButton = (Button) findViewById(R.id.nav_button);
	        navButton.setOnClickListener(new View.OnClickListener() {
	            @Override
	            public void onClick(View v) {
	                Intent intent = new Intent(WeatherActivity.this,
	                        CountyChoosedActivity.class);
	                startActivity(intent);
	            }
	        });
	
	        titleCityText = (TextView) findViewById(R.id.title_city);
	        titleCityText.setText(curCountyName);
	
	        //初始化pager，每个页面都是一个城市的天气
	        viewPager = (ViewPager) findViewById(R.id.weather_pager);
	        LayoutInflater layoutInflater = getLayoutInflater();
	        for (int i=0;i<choosedCountyList.size(); i++){
	            View view = layoutInflater.inflate(R.layout.weather_layout, null);
	            view.setTag(i);
	            String countyName = choosedCountyList.get(i);
	            fillView(view, countyName,true);
	            pageList.add(view);
	        }
	        adapter = new PagerAdapter() {
	            @Override
	            public int getCount() {
	                return pageList.size();
	            }
	
	            @Override
	            public int getItemPosition(@NonNull Object object) {
	                if (updateStatu == UPDATE_ONE_PAGE){
	                    //更新其中一个页面
	                    View view = (View) object;
	                    int currentPagerIdx = viewPager.getCurrentItem();
	                    if (currentPagerIdx == (Integer) view.getTag()) {
	                        return POSITION_NONE;
	                    } else {
	                        return POSITION_UNCHANGED;
	                    }
	                }
	                else if (updateStatu == UPDATE_ALL_PAGE){
	                    //更新所有页面，目的为去除缓存
	                    return POSITION_NONE;
	                }
	                return POSITION_NONE;
	            }
	
	            @Override
	            public boolean isViewFromObject(@NonNull View view, @NonNull Object object) {
	                return view==object;
	            }
	
	            @Override
	            public void destroyItem(@NonNull ViewGroup container, int position, @NonNull Object object) {
	                container.removeView((View) object);
	            }
	
	            @NonNull
	            @Override
	            public Object instantiateItem(@NonNull ViewGroup container, int position) {
	                container.addView(pageList.get(position));
	                return pageList.get(position);
	            }
	        };
	        viewPager.setAdapter(adapter);
	        //侦测当前页面位置
	        viewPager.setOnPageChangeListener(new ViewPager.OnPageChangeListener() {
	            @Override
	            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
	            }
	
	            @Override
	            public void onPageSelected(int position) {
	                WeatherActivity.this.position = position;
	                curCountyName = choosedCountyList.get(position);
	                titleCityText.setText(curCountyName);
	                for (int i=0;i<dotImageList.size(); i++){
	                    dotImageList.get(i).setBackgroundResource(R.drawable.dot_normal);
	                }
	                dotImageList.get(position).setBackgroundResource(R.drawable.dot_focused);
	            }
	
	            @Override
	            public void onPageScrollStateChanged(int state) {
	
	            }
	        });
	
	        viewPager.setCurrentItem(position);
	        viewPager.setOffscreenPageLimit(8);
	
	    }
	
	    private int getPositionFromName(String name,List<String> list){
	        for (int i = 0; i < list.size(); i++) {
	            if (name.equals(list.get(i))){
	                return i;
	            }
	        }
	        return -1;
	    }
	
	    public  boolean equalList(List<String> a, List<String> b) {
	        if (a==b) return true;
	        if (a==null || b==null) return false;
	        int length = a.size();
	        if (b.size() != length) return false;
	        for (int i=0; i<length; i++)
	            if (!a.get(i).equals(b.get(i))) return false;
	        return true;
	    }
	
	    /**
	     * 当重新激活后，
	     * 1)比较选择城市是否发生改变，若无，显示当前选中的城市页面
	     * 2 发生改变，添加或减少页面
	     */
	    @Override
	    protected void onRestart() {
	        super.onRestart();
	        //当前选择的城市列表
	        List<String> curList = initChoosedCounties();
	        curCountyName = prefs.getString("curCounty", curList.get(0));
	        position = getPositionFromName(curCountyName, curList);
	        //改变圆点位置
	        dotImageList.clear();
	        dotLayout.removeAllViews();
	        for (int i = 0; i < curList.size(); i++) {
	            ImageView imageView = new ImageView(WeatherActivity.this);
	            imageView.setLayoutParams(new ViewGroup.LayoutParams(30, 30));
	            imageView.setPadding(20, 0, 20, 0);
	            dotImageList.add(imageView);
	            dotLayout.addView(imageView);
	        }
	
	        //当前变化
	        if (!equalList(curList, choosedCountyList)){
	            LayoutInflater layoutInflater = getLayoutInflater();
	            pageList.clear();
	            for (int i=0; i< curList.size();i++){
	                //以前的没有，新增的城市，新增页面
	                View view = layoutInflater.inflate(R.layout.weather_layout, null);
	                view.setTag(i);
	                String countyName = curList.get(i);
	                fillView(view, countyName,false);
	                pageList.add(i, view);
	            }
	            updateStatu = UPDATE_ALL_PAGE;
	            adapter.notifyDataSetChanged();
	
	        }
	        //一定要在选择页面前更新城市列表，不然增加城市时onPageSelected出现超过索引的问题
	        choosedCountyList = curList;
	        viewPager.setCurrentItem(position);
	    }
	
	    private List<String> initChoosedCounties(){
	        List<ChoosedCounty> all = DataSupport.findAll(ChoosedCounty.class);
	        List<String> curList = new ArrayList<>();
	        for (int i=0; i<all.size(); i++){
	            curList.add(all.get(i).getCountyName());
	        }
	        return curList;
	    }
	
	    /**
	     * 根据城市名获取内容，并填充view
	     * @param countyName
	     * @param isUpdate 是否更新天气数据
	     */
	    private void fillView(View view, String countyName, boolean isUpdate){
	        if (isUpdate){
	            requestWeather(view,countyName);
	        }else {
	            String weatherString = prefs.getString(countyName, null);
	            Weather weather = null;
	            if (weatherString == null){
	                // 无缓存时去服务器查询天气
	                requestWeather(view, countyName);
	            }else {
	                weather = Utility.handleWeatherResponse(weatherString);
	                showWeatherInfo(view, weather);
	            }
	        }
	
	    }
	
	    /**
	     * 根据城市名来查询天气
	     * 支持中英文与拼音，
	     * @param city
	     */
	    public void requestWeather(final View view, final String city){
	        String weatherUrl = "https://free-api.heweather.net/s6/weather/?location="
	                +city+"&key=自己和风天气的key";
	        HttpUtil.sendOkHttpRequest(weatherUrl, new Callback() {
	            @Override
	            public void onFailure(Call call, IOException e) {
	                e.printStackTrace();
	                runOnUiThread(new Runnable() {
	                    @Override
	                    public void run() {
	                        Toast.makeText(WeatherActivity.this, "获取天气信息失败",
	                                Toast.LENGTH_SHORT).show();
	                        swipeRefreshLayout.setRefreshing(false);
	                    }
	                });
	            }
	
	            @Override
	            public void onResponse(Call call, Response response) throws IOException {
	                final String responseText = response.body().string();
	                final Weather weather = Utility.handleWeatherResponse(responseText);
	                runOnUiThread(new Runnable() {
	                    @Override
	                    public void run() {
	                        if (weather!=null && "ok".equals(weather.status)){
	                            @SuppressLint("CommitPrefEdits")
	                            SharedPreferences.Editor edit = prefs.edit();
	                            edit.putString(city, responseText);
	                            edit.apply();
	                            showWeatherInfo(view, weather);
	                        }else {
	                            Toast.makeText(WeatherActivity.this,"获取天气信息失败",
	                                    Toast.LENGTH_SHORT).show();
	                        }
	                        swipeRefreshLayout.setRefreshing(false);
	                    }
	                });
	            }
	        });
	        loadBingPIc();
	    }
	
	    /**
	     * 处理并展示Weather实体类数据信息
	     * @param weather
	     */
	    private void showWeatherInfo(View view, Weather weather){
	        // 初始化各控件
	        ScrollView weatherLayout = (ScrollView) view.findViewById(R.id.weather_layout);
	        //解决SwipeRefreshLayout与ScrollView的下拉冲突
	        weatherLayout.setOnScrollChangeListener(new View.OnScrollChangeListener() {
	            @Override
	            public void onScrollChange(View v, int scrollX, int scrollY, int oldScrollX, int oldScrollY) {
	                if (swipeRefreshLayout != null){
	                    swipeRefreshLayout.setEnabled(scrollY==0);
	                }
	            }
	        });
	        TextView updateTimeText = (TextView) view.findViewById(R.id.update_time);
	        TextView degreeText = (TextView) view.findViewById(R.id.degree_text);
	        TextView weatherInfoText = (TextView) view.findViewById(R.id.weather_info_text);
	        LinearLayout forecastLayout = (LinearLayout) view.findViewById(R.id.forecast_layout);
	        LinearLayout lifestyleLayout  = (LinearLayout) view.findViewById(R.id.lifestyle_layout);
	        String updateTime = weather.update.loc;
	        String degree = weather.now.tmp+"℃";
	        String weatherInfo = weather.now.cond_txt;
	        //填充内容
	        updateTimeText.setText(updateTime);
	        degreeText.setText(degree);
	        weatherInfoText.setText(weatherInfo);
	        forecastLayout.removeAllViews();
	        for (Forecast forecast:weather.daily_forecast){
	            View viewN = LayoutInflater.from(this).inflate(R.layout.forecast_item,
	                    forecastLayout, false);
	            TextView dateText = (TextView) viewN.findViewById(R.id.date_text);
	            TextView infoText = (TextView) viewN.findViewById(R.id.info_text);
	            TextView minText = (TextView) viewN.findViewById(R.id.min_text);
	            TextView maxText = (TextView) viewN.findViewById(R.id.max_text);
	
	            dateText.setText(forecast.date);
	            infoText.setText(forecast.cond_txt_d);
	            minText.setText(forecast.tmp_min);
	            maxText.setText(forecast.tmp_max);
	            forecastLayout.addView(viewN);
	        }
	        lifestyleLayout.removeAllViews();
	        for (Lifestyle lifestyle:weather.lifestyles){
	            View viewN = LayoutInflater.from(this).inflate(R.layout.lifestyle_item,
	                    lifestyleLayout, false);
	            TextView lifestyle_text_text = (TextView) viewN.findViewById(R.id.lifestyle_text_text);
	            TextView lifestyle_type_text = (TextView) viewN.findViewById(R.id.lifestyle_type_text);
	            lifestyle_type_text.setText(Utility.translate(lifestyle.type) +" "+ lifestyle.brf);
	            lifestyle_text_text.setText(lifestyle.txt);
	            lifestyleLayout.addView(viewN);
	        }
	        weatherLayout.setVisibility(View.VISIBLE);
	    }
	
	    /**
	     * 更新当前页天气
	     */
	    private void updatePageView(){
	        updateStatu = UPDATE_ONE_PAGE;
	        View view = pageList.get(position);
	        String countyName = choosedCountyList.get(position);
	        fillView(view, countyName, true);
	        adapter.notifyDataSetChanged();
	    }
	
	    private void loadBingPIc(){
	        String requestBingPicUrl = "http://guolin.tech/api/bing_pic";
	        HttpUtil.sendOkHttpRequest(requestBingPicUrl, new Callback() {
	            @Override
	            public void onFailure(Call call, IOException e) {
	                e.printStackTrace();
	            }
	
	            @Override
	            public void onResponse(Call call, Response response) throws IOException {
	                final String pic = response.body().string();
	                SharedPreferences.Editor edit = PreferenceManager.getDefaultSharedPreferences(WeatherActivity.this).edit();
	                edit.putString("bing_pic", pic);
	                edit.apply();
	                runOnUiThread(new Runnable() {
	                    @Override
	                    public void run() {
	                        Glide.with(WeatherActivity.this).load(pic).into(bingPIcImg);
	                    }
	                });
	            }
	        });
	
	
	    }
	
	}


CountyChoosedActivity
展示已选中的城市列表和删除城市列表

    package com.coolweather.android;

	import androidx.appcompat.app.AlertDialog;
	import androidx.appcompat.app.AppCompatActivity;
	import androidx.fragment.app.Fragment;
	import androidx.fragment.app.FragmentManager;
	import androidx.fragment.app.FragmentTransaction;
	
	import android.annotation.SuppressLint;
	import android.content.DialogInterface;
	import android.content.Intent;
	import android.content.SharedPreferences;
	import android.os.Bundle;
	import android.preference.PreferenceManager;
	import android.util.ArraySet;
	import android.util.Log;
	import android.view.View;
	import android.widget.AdapterView;
	import android.widget.ArrayAdapter;
	import android.widget.Button;
	import android.widget.ListView;
	import android.widget.TextView;
	import android.widget.Toast;
	
	import com.baidu.location.BDLocation;
	import com.baidu.location.BDLocationListener;
	import com.baidu.location.LocationClient;
	import com.baidu.location.LocationClientOption;
	import com.coolweather.android.db.ChoosedCounty;
	import com.coolweather.android.util.BDLocationUtil;
	import com.coolweather.android.util.Utility;
	
	import org.litepal.crud.DataSupport;
	
	import java.util.ArrayList;
	import java.util.HashSet;
	import java.util.List;
	
	import static org.litepal.LitePalApplication.getContext;
	
	public class CountyChoosedActivity extends AppCompatActivity {
	
	    private static final String TAG = "CountyChoosedActivity";
	    private SharedPreferences prefs;
	    private Button addCountyButton;
	    private ListView listView;
	    private Button locationCityButton;
	    private ArrayAdapter<String> adapter;
	    private List<String> countyList = new ArrayList<>();
	    private List<ChoosedCounty> countyListDB = new ArrayList<>();
	
	    private LocationClient locationClient;
	
	    @Override
	    protected void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_county_choosed);
	        prefs = PreferenceManager.getDefaultSharedPreferences(this);
	
	        queryChoosedCountyList();
	        addCountyButton = (Button) findViewById(R.id.add_county_button);
	        listView = (ListView) findViewById(R.id.county_list_choosed_view);
	        locationCityButton = (Button) findViewById(R.id.location_city_button);
	
	        startLocation();
	
	        addCountyButton.setOnClickListener(new View.OnClickListener() {
	            @Override
	            public void onClick(View v) {
	                Intent intent = new Intent(getContext(), ChooseAreaActivity.class);
	                startActivity(intent);
	                finish();
	            }
	        });
	
	        adapter = new ArrayAdapter<String>(CountyChoosedActivity.this,
	                android.R.layout.simple_list_item_1, countyList);
	        listView.setAdapter(adapter);
	        listView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
	            @Override
	            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
	                String s = countyList.get(position);
	                SharedPreferences.Editor edit = prefs.edit();
	                edit.putString("curCounty", s);
	                edit.apply();
	                Intent intent = new Intent(getContext(), WeatherActivity.class);
	                startActivity(intent);
	                finish();
	            }
	        });
	        listView.setOnItemLongClickListener(new AdapterView.OnItemLongClickListener() {
	            @Override
	            public boolean onItemLongClick(AdapterView<?> parent, View view, final int position, long id) {
	                //定义AlertDialog.Builder对象，当长按列表项的时候弹出确认删除对话框
	                AlertDialog.Builder builder=new AlertDialog.Builder(CountyChoosedActivity.this);
	                builder.setMessage("确定删除?");
	                builder.setTitle("提示");
	
	                //添加AlertDialog.Builder对象的setPositiveButton()方法
	                builder.setPositiveButton("确定", new DialogInterface.OnClickListener() {
	                    @Override
	                    public void onClick(DialogInterface dialog, int which) {
	                        String countyName= countyList.get(position);
	                        if(countyList.remove(position)!=null){
	                            DataSupport.deleteAll(ChoosedCounty.class,
	                                    "countyName = ?", countyName);
	                            System.out.println("success");
	                        }else {
	                            System.out.println("failed");
	                        }
	                        adapter.notifyDataSetChanged();
	                        Toast.makeText(getBaseContext(), "删除列表项", Toast.LENGTH_SHORT).show();
	                    }
	                });
	
	                //添加AlertDialog.Builder对象的setNegativeButton()方法
	                builder.setNegativeButton("取消", new DialogInterface.OnClickListener() {
	                    @Override
	                    public void onClick(DialogInterface dialog, int which) {
	
	                    }
	                });
	
	                AlertDialog alertDialog = builder.create();
	                alertDialog.setCanceledOnTouchOutside(false);
	                alertDialog.show();
	                return true;
	
	            }
	        });
	
	    }
	
	    private void startLocation(){
	        locationClient = new LocationClient(getApplicationContext());
	        locationClient.registerLocationListener(new BDLocationListener() {
	            @Override
	            public void onReceiveLocation(final BDLocation location) {
	                String district = location.getDistrict();
	                if (district == null){
	                    runOnUiThread(new Runnable() {
	                        @Override
	                        public void run() {
	                            Toast.makeText(CountyChoosedActivity.this,
	                                    "无法获取定位，重置定位为北京",
	                                    Toast.LENGTH_SHORT).show();
	                        }
	                    });
	                    district = "北京";
	                    locationCityButton.setText( district+"--重新定位");
	                }else {
	                    locationCityButton.setText( district+"--定位");
	                }
	                locationClient.stop();
	                final String finalDistrict = district;
	                locationCityButton.setOnClickListener(new View.OnClickListener() {
	                    @Override
	                    public void onClick(View v) {
	                        SharedPreferences.Editor edit = prefs.edit();
	                        Utility.saveChoosedCounty(finalDistrict);
	                        edit.putString("curCounty", finalDistrict);
	                        edit.apply();
	                        Intent intent = new Intent(CountyChoosedActivity.this, WeatherActivity.class);
	                        startActivity(intent);
	                        finish();
	                    }
	                });
	            }
	        });
	        LocationClientOption option = new LocationClientOption();
	        option.setIsNeedAddress(true);
	        locationClient.setLocOption(option);
	        locationClient.start();
	    }
	
	
	    private void queryChoosedCountyList(){
	        countyListDB = DataSupport.findAll(ChoosedCounty.class);
	        for (ChoosedCounty county:countyListDB){
	            countyList.add(county.getCountyName());
	        }
	    }
	
	
	}

ChooseAreaActivity和ChooseAreaFragment，用来新增城市
	
	package com.coolweather.android;

	import androidx.annotation.Nullable;
	import androidx.appcompat.app.AppCompatActivity;
	
	import android.os.Bundle;
	
	
	public class ChooseAreaActivity extends AppCompatActivity {
	    @Override
	    protected void onCreate(@Nullable Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.activity_choose_area);
	    }
	
	}

-

	package com.coolweather.android;

	import android.app.ProgressDialog;
	import android.content.Intent;
	import android.content.SharedPreferences;
	import android.os.Bundle;
	import android.preference.PreferenceManager;
	import android.view.LayoutInflater;
	import android.view.View;
	import android.view.ViewGroup;
	import android.widget.AdapterView;
	import android.widget.ArrayAdapter;
	import android.widget.Button;
	import android.widget.ListView;
	import android.widget.TextView;
	import android.widget.Toast;
	
	import androidx.annotation.NonNull;
	import androidx.annotation.Nullable;
	import androidx.fragment.app.Fragment;
	
	import com.coolweather.android.db.City;
	import com.coolweather.android.db.County;
	import com.coolweather.android.db.Province;
	import com.coolweather.android.util.HttpUtil;
	import com.coolweather.android.util.Utility;
	
	import org.litepal.crud.DataSupport;
	
	import java.io.IOException;
	import java.util.ArrayList;
	import java.util.List;
	
	import okhttp3.Call;
	import okhttp3.Callback;
	import okhttp3.Response;
	
	public class ChooseAreaFragment extends Fragment {
	
	    private SharedPreferences prefs;
	    public static final int LEVEL_PROVINCE = 0;
	    public static final int LEVEL_CITY = 1;
	    public static final int LEVEL_COUNTY = 2;
	    private ProgressDialog progressDialog;
	    private TextView titleText;
	    private Button backButton;
	    private ListView listView;
	    private ArrayAdapter<String> adapter;
	    private List<String> dataList = new ArrayList<>();
	    /**
	     * 省列表
	     */
	    private List<Province> provinceList;
	    /**
	     * 市列表
	     */
	    private List<City> cityList;
	    /**
	     * 县列表
	     */
	    private List<County> countyList;
	
	    /**
	     * 选中的省份
	    */
	    private Province selectedProvince;
	    /**
	     * 选中的城市
	     */
	    private City selectedCity;
	    /**
	     * 当前选中的级别
	     */
	    private int currentLevel;
	
	    @Nullable
	    @Override
	    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
	        prefs = PreferenceManager.getDefaultSharedPreferences(getContext());
	        View view = inflater.inflate(R.layout.choose_area, container, false);
	        titleText = (TextView)view.findViewById(R.id.title_text);
	        backButton = (Button) view.findViewById(R.id.back_button);
	        listView = (ListView) view.findViewById(R.id.list_view);
	
	        adapter = new ArrayAdapter<>(getContext(), android.R.layout.simple_list_item_1, dataList);
	        listView.setAdapter(adapter);
	        return view;
	    }
	
	    @Override
	    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
	        super.onActivityCreated(savedInstanceState);
	        listView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
	            @Override
	            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
	                if (currentLevel == LEVEL_PROVINCE){
	                    selectedProvince = provinceList.get(position);
	                    queryCities();
	                }else if (currentLevel == LEVEL_CITY){
	                    selectedCity = cityList.get(position);
	                    queryCounties();
	                }else if (currentLevel== LEVEL_COUNTY){
	                    String countyName = countyList.get(position).getCountyName();
	                    Intent intent = new Intent(getActivity(), WeatherActivity.class);
	                    //保存添加的城市
	                    Utility.saveChoosedCounty(countyName);
	                    SharedPreferences.Editor edit = prefs.edit();
	                    edit.putString("curCounty", countyName);
	                    edit.apply();
	                    startActivity(intent);
	                    getActivity().finish();
	
	                }
	            }
	        });
	
	        backButton.setOnClickListener(new View.OnClickListener() {
	            @Override
	            public void onClick(View v) {
	                if (currentLevel == LEVEL_COUNTY) {
	                    queryCities();
	                } else if (currentLevel == LEVEL_CITY) {
	                    queryProvinces();
	                }else if(currentLevel == LEVEL_PROVINCE){
	                    Intent intent = new Intent(getActivity(), CountyChoosedActivity.class);
	                    startActivity(intent);
	                    getActivity().finish();
	                }
	            }
	        });
	        //初始化省份数据
	        queryProvinces();
	    }
	
	    /**
	     * 查询全国所有的省， 优先从数据库查询， 如果没有查询到再去服务器上查询
	    */
	    private void queryProvinces() {
	        titleText.setText("中国");
	        backButton.setVisibility(View.VISIBLE);
	        provinceList = DataSupport.findAll(Province.class);
	        if (provinceList.size() > 0) {
	            dataList.clear();
	            for (Province province : provinceList) {
	                dataList.add(province.getProvinceName());
	            }
	            adapter.notifyDataSetChanged();
	            listView.setSelection(0);
	            currentLevel = LEVEL_PROVINCE;
	        } else {
	            String address = "http://guolin.tech/api/china";
	            queryFromServer(address, "province");
	        }
	    }
	
	    /**
	     * 查询选中省内所有的市， 优先从数据库查询， 如果没有查询到再去服务器上查询
	    */
	    private void queryCities() {
	        titleText.setText(selectedProvince.getProvinceName());backButton.setVisibility(View.VISIBLE);
	        cityList = DataSupport.where("provinceid = ?",
	                String.valueOf(selectedProvince.getId())).find(City.class);
	        if (cityList.size() > 0) {
	            dataList.clear();
	            for (City city : cityList) {
	                dataList.add(city.getCityName());
	            }
	            adapter.notifyDataSetChanged();
	            listView.setSelection(0);
	            currentLevel = LEVEL_CITY;
	        } else {
	            int provinceCode = selectedProvince.getProvinceCode();
	            String address = "http://guolin.tech/api/china/" + provinceCode;
	            queryFromServer(address, "city");
	        }
	    }
	
	    /**
	     * 查询选中市内所有的县， 优先从数据库查询， 如果没有查询到再去服务器上查询
	    */
	    private void queryCounties() {
	        titleText.setText(selectedCity.getCityName());
	        backButton.setVisibility(View.VISIBLE);
	        countyList = DataSupport.where("cityid = ?",
	                String.valueOf(selectedCity.
	                getId())).find(County.class);
	        if (countyList.size() > 0) {
	            dataList.clear();
	            for (County county : countyList) {
	                dataList.add(county.getCountyName());
	            }
	            adapter.notifyDataSetChanged();
	            listView.setSelection(0);
	            currentLevel = LEVEL_COUNTY;
	        } else {
	            int provinceCode = selectedProvince.getProvinceCode();
	            int cityCode = selectedCity.getCityCode();
	            String address = "http://guolin.tech/api/china/" + provinceCode + "/" +
	                    cityCode;
	            queryFromServer(address, "county");
	        }
	    }
	
	    /**
	     * 根据传入的地址和类型从服务器上查询省市县数据
	    */
	    private void queryFromServer(String address, final String type) {
	        showProgressDialog();
	        HttpUtil.sendOkHttpRequest(address, new Callback() {
	            @Override
	            public void onResponse(Call call, Response response) throws IOException {
	                String responseText = response.body().string();
	                boolean result = false;
	                if ("province".equals(type)) {
	                    result = Utility.handleProvinceResponse(responseText);
	                } else if ("city".equals(type)) {
	                    result = Utility.handleCityResponse(responseText,
	                            selectedProvince.getId());
	                } else if ("county".equals(type)) {
	                    result = Utility.handleCountyResponse(responseText,
	                            selectedCity.getId());
	                }
	                if (result) {
	                    getActivity().runOnUiThread(new Runnable() {@Override
	                    public void run() {
	                        closeProgressDialog();
	                        if ("province".equals(type)) {
	                            queryProvinces();
	                        } else if ("city".equals(type)) {
	                            queryCities();
	                        } else if ("county".equals(type)) {
	                            queryCounties();
	                        }
	                    }
	                    });
	                }
	            }
	
	            @Override
	            public void onFailure(Call call, IOException e) {
	            // 通过runOnUiThread()方法回到主线程处理逻辑
	                getActivity().runOnUiThread(new Runnable() {
	                    @Override
	                    public void run() {
	                        closeProgressDialog();
	                        Toast.makeText(getContext(), "加载失败", Toast.LENGTH_SHORT).
	                                show();
	                    }
	                });
	            }
	        });
	    }
	
	    /**
	     * 显示进度对话框
	    */
	    private void showProgressDialog() {
	        if (progressDialog == null) {
	            progressDialog = new ProgressDialog(getActivity());
	            progressDialog.setMessage("正在加载...");
	            progressDialog.setCanceledOnTouchOutside(false);
	        }
	        progressDialog.show();
	    }
	    /**
	     * 关闭进度对话框
	    */
	    private void closeProgressDialog() {
	        if (progressDialog != null) {
	            progressDialog.dismiss();
	        }
	    }
	}


AutoUpdateService后台服务，用来自动更新天气

	package com.coolweather.android;
	
	import android.app.AlarmManager;
	import android.app.PendingIntent;
	import android.app.Service;
	import android.content.Intent;
	import android.content.SharedPreferences;
	import android.os.IBinder;
	import android.os.SystemClock;
	import android.preference.PreferenceManager;
	
	import com.bumptech.glide.Glide;
	import com.coolweather.android.db.ChoosedCounty;
	import com.coolweather.android.gson.Weather;
	import com.coolweather.android.util.BDLocationUtil;
	import com.coolweather.android.util.HttpUtil;
	import com.coolweather.android.util.Utility;
	
	import org.litepal.crud.DataSupport;
	
	import java.io.IOException;
	import java.util.List;
	
	import okhttp3.Call;
	import okhttp3.Callback;
	import okhttp3.Response;
	
	public class AutoUpdateService extends Service {
	    public AutoUpdateService() {
	    }
	
	    @Override
	    public IBinder onBind(Intent intent) {
	        return null;
	    }
	
	    @Override
	    public int onStartCommand(Intent intent, int flags, int startId) {
	        updateBingPic();
	        //更新天气
	        String districtLocation = BDLocationUtil.getLocationDistrict(getApplicationContext());
	        updateWeather(districtLocation);
	        List<ChoosedCounty> list = DataSupport.findAll(ChoosedCounty.class);
	        for(ChoosedCounty county:list){
	            updateWeather(county.getCountyName());
	        }
	
	        AlarmManager manager = (AlarmManager) getSystemService(ALARM_SERVICE);
	        int time = 8 * 60 * 60 * 1000;
	        long triggerTime = SystemClock.elapsedRealtime()+time;
	        Intent intentA = new Intent(this, AutoUpdateService.class);
	        PendingIntent pendingIntent = PendingIntent.getService(this, 0, intentA, 0);
	        manager.cancel(pendingIntent);
	        manager.set(AlarmManager.ELAPSED_REALTIME_WAKEUP, triggerTime, pendingIntent);
	        return super.onStartCommand(intent, flags, startId);
	    }
	
	    private void updateWeather(final String name){
	        SharedPreferences pref = PreferenceManager.getDefaultSharedPreferences(this);
			//和风天气
	        String weatherUrl =  "https://free-api.heweather.net/s6/weather/?location="
	                +name+"&key=自己的key";
	        HttpUtil.sendOkHttpRequest(weatherUrl, new Callback() {
	            @Override
	            public void onFailure(Call call, IOException e) {
	                e.printStackTrace();
	            }
	
	            @Override
	            public void onResponse(Call call, Response response) throws IOException {
	                String reponseText = response.body().string();
	                Weather weather = Utility.handleWeatherResponse(reponseText);
	                if (weather!=null  && "ok".equals(weather.status)){
	                    SharedPreferences.Editor edit = PreferenceManager.
	                            getDefaultSharedPreferences(AutoUpdateService.this).edit();
	                    edit.putString(name, reponseText);
	                    edit.apply();
	                }
	            }
	        });
	    }
	
	    private void updateBingPic(){
	        String requestBingPicUrl = "http://guolin.tech/api/bing_pic";
	        HttpUtil.sendOkHttpRequest(requestBingPicUrl, new Callback() {
	            @Override
	            public void onFailure(Call call, IOException e) {
	                e.printStackTrace();
	            }
	
	            @Override
	            public void onResponse(Call call, Response response) throws IOException {
	                final String pic = response.body().string();
	                SharedPreferences.Editor edit = PreferenceManager.getDefaultSharedPreferences(AutoUpdateService.this).edit();
	                edit.putString("bing_pic", pic);
	                edit.apply();
	            }
	        });
	    }
	}


Utility和HttpUtil工具类

    package com.coolweather.android.util;

	import android.text.TextUtils;
	import com.coolweather.android.db.ChoosedCounty;
	import com.coolweather.android.db.City;
	import com.coolweather.android.db.County;
	import com.coolweather.android.db.Province;
	import com.coolweather.android.gson.Weather;
	import com.google.gson.Gson;
	
	import org.json.JSONArray;
	import org.json.JSONException;
	import org.json.JSONObject;
	import org.litepal.crud.DataSupport;
	
	import java.util.HashMap;
	import java.util.List;
	import java.util.Map;
	
	public class Utility {
	
	    /**
	     * 解析处理服务器返回的省级数据
	     */
	    public static boolean handleProvinceResponse(String response){
	        if (!TextUtils.isEmpty(response)){
	            try{
	                JSONArray allProvinces = new JSONArray(response);
	                for (int i = 0; i < allProvinces.length(); i++) {
	                    JSONObject provinceObject = allProvinces.getJSONObject(i);
	                    Province province = new Province();
	                    province.setProvinceName(provinceObject.getString("name"));
	                    province.setProvinceCode(provinceObject.getInt("id"));
	                    province.save();
	                }
	                return true;
	            } catch (JSONException e) {
	                e.printStackTrace();
	            }
	
	        }
	        return false;
	    }
	
	    /**
	     * 解析处理服务器返回的市ji数据
	     * @param response
	     * @param provinceId
	     * @return
	     */
	    public static boolean handleCityResponse(String response, int provinceId){
	        if (!TextUtils.isEmpty(response)) {
	            try {
	                JSONArray allCities = new JSONArray(response);
	                for (int i = 0; i < allCities.length(); i++) {
	                    JSONObject cityObject = allCities.getJSONObject(i);
	                    City city = new City();
	                    city.setCityName(cityObject.getString("name"));
	                    city.setCityCode(cityObject.getInt("id"));
	                    city.setProvinceId(provinceId);
	                    city.save();
	                }
	                return true;
	            } catch (JSONException e) {
	                e.printStackTrace();
	            }
	        }
	        return false;
	    }
	
	    /**
	     * 解析和处理服务器返回的县级数据
	    */
	    public static boolean handleCountyResponse(String response, int cityId) {
	        if (!TextUtils.isEmpty(response)) {
	            try {
	                JSONArray allCounties = new JSONArray(response);
	                for (int i = 0; i < allCounties.length(); i++) {
	                    JSONObject countyObject = allCounties.getJSONObject(i);
	                    County county = new County();
	                    county.setCountyName(countyObject.getString("name"));
	                    county.setWeatherId(countyObject.getString("weather_id"));
	                    county.setCityId(cityId);
	                    county.save();
	                }
	                return true;
	            } catch (JSONException e) {
	                e.printStackTrace();
	            }
	        }
	        return false;
	    }
	
	    public static Weather handleWeatherResponse(String response){
	        try {
	            JSONObject jsonObject = new JSONObject(response);
	            JSONArray jsonArray = jsonObject.getJSONArray("HeWeather6");
	            String weatherContent = jsonArray.getJSONObject(0).toString();
	            return new Gson().fromJson(weatherContent, Weather.class);
	        } catch (Exception e) {
	            e.printStackTrace();
	        }
	        return null;
	    }
	
	    private static final Map<String, String> myMap;
	    static
	    {
	        myMap = new HashMap<String, String>();
	        myMap.put("comf","舒适度指数");
	        myMap.put("cw","洗车指数");
	        myMap.put("drsg","穿衣指数");
	        myMap.put("flu","感冒指数");
	        myMap.put("ptfc","交通指数");
	        myMap.put("trav","旅游指数");
	        myMap.put("sport","运动指数");
	        myMap.put("uv","紫外线指数");
	        myMap.put("air","空气污染扩散条件指数");
	        myMap.put("ac","空调开启指数");
	        myMap.put("ag","过敏指数");
	        myMap.put("gl","太阳镜指数");
	        myMap.put("mu","化妆指数");
	        myMap.put("airc","晾晒指数");
	        myMap.put("fsh","钓鱼指数");
	        myMap.put("spi","防晒指数");
	    }
	
	    public static String translate(String type){
	        return myMap.get(type);
	    }
	
	
	    public static void saveChoosedCounty(String countyName){
	        //去重
	        List<ChoosedCounty> all = DataSupport.findAll(ChoosedCounty.class);
	        for (ChoosedCounty county: all){
	            if (countyName.equals(county.getCountyName())){
	                return;
	            }
	        }
	        ChoosedCounty county = new ChoosedCounty(countyName);
	        county.save();
	    }
	
	}

	package com.coolweather.android.util;

	import okhttp3.OkHttpClient;
	import okhttp3.Request;
	
	public class HttpUtil {
	
	    public static void sendOkHttpRequest(String address, okhttp3.Callback callback){
	        OkHttpClient client = new OkHttpClient();
	        Request request = new Request.Builder().url(address).build();
	        client.newCall(request).enqueue(callback);
	    }
	}


*[项目的github地址](https://github.com/BARKTEGH/CoolWeather "Github地址")*