---
title: '自定义View实现天气折线图效果'
date: 2016-06-02 00:00:40
tags: [Custom View]
categories: [Android,Custom View]
---

### 前言
最近公司的项目加入天气模块，需要实现下面的效果。
![设计图](/images/weather_design.png)
然后根据自己的构想实现了下面的效果。
![实现的效果图](/images/weather_implement.gif)

### 构想思路
其实在拿到设计的一个效果，我们首先要做的就是去思考，怎么实现，就算不好实现，也要实现一个折中的两边都可以妥协的方案。

由于当前是要展示10天以上的天气的情况，那么如果采用一个view绘制的形式肯定会影响到性能，那其实很快就想到了ListView，这不就是一个横向的ListView的效果么，考虑到ListView并没有横向的效果，转而就想到了RecyclerView，RecyclerView的LinearLayoutManager可以直接设置为横向的，这解决了使用什么来实现的问题。

#### 最高温度和最低温度在View上面怎么绘制
这个问题不是太难，可以这样，拿到这15天的最低温度和最高温度，这个需要我们计算，从这15个最高以及最低温度里面找到最大以及最小的，然后使用这两个温度差去映射View的高度，最后可以得到需要的绘制的圆点View的Y轴的计算公式。
$$需要在View上面绘制的高度 = \frac{(今天的温度-15天里面最低温度)View的高度}{15天里面最高温度-15天里面最低温度} $$

#### 折线在每个view的最左边个最右边的位置
这个也很简单，用下面的公式即可。
 $$\frac{昨天的温度+今天的温度}{2}$$
然后再经过上面的计算就可以得到需要绘制的Y轴的位置。

<!-- more -->

### 具体实现
#### WeatherLineView的实现
自定义WeatherLineView，需要下面的几个属性:
```xml
<declare-styleable name="WeatherLineView">
    <!-- 文字大小 -->
    <attr name="temperTextSize" format="dimension"/>
    <!-- 文字的颜色 -->
    <attr name="weatextColor" format="color"/>
    <!-- 绘制的线的宽度 -->
    <attr name="weaLineWidth" format="dimension"/>
    <!-- 绘制的圆点的半径 -->
    <attr name="weadotRadius" format="dimension"/>
    <!-- 文字离圆点的距离 -->
    <attr name="textDotDistance" format="dimension"/>
</declare-styleable>
```
具体的实现:
```java
public class WeatherLineView extends View {

    /**
     * 默认最小宽度50dp
     */
    private static final int defaultMinWidth = 100;

    /**
     * 默认最小高度80dp
     */
    private static final int defaultMinHeight = 80;

    /**
     * 字体最小默认16dp
     */
    private int mTemperTextSize = 16;

    /**
     * 文字颜色
     */
    private int mWeaTextColor = Color.BLACK;

    /**
     * 线的宽度
     */
    private int mWeaLineWidth = 1;

    /**
     * 圆点的宽度
     */
    private int mWeaDotRadius = 5;

    /**
     * 文字和点的间距
     */
    private int mTextDotDistance = 5;

    /**
     * 画文字的画笔
     */
    private TextPaint mTextPaint;

    /**
     * 文字的FontMetrics
     */
    private Paint.FontMetrics mTextFontMetrics;

    /**
     * 画点的画笔
     */
    private Paint mDotPaint;

    /**
     * 画线的画笔
     */
    private Paint mLinePaint;

    /**
     * 15天最低温度的数据
     */
    private int mLowestTemperData;

    /**
     * 15天最高温度的数据
     */
    private int mHighestTemperData;

    /**
     * 分别代表最左边的，中间的，右边的三个当天最低温度值
     */
    private int mLowTemperData[];

    private int mHighTemperData[];

    public WeatherLineView(Context context) {
        this(context, null);
    }

    public WeatherLineView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public WeatherLineView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context, attrs, defStyleAttr);
        initPaint();
    }

    /**
     * 设置当天的三个低温度数据，中间的数据就是当天的最低温度数据，
     * 第一个数据是当天和前天的数据加起来的平均数，
     * 第二个数据是当天和明天的数据加起来的平均数
     *
     * @param low  最低温度
     * @param high 最高温度
     */
    public void setLowHighData(int low[], int high[]) {
        mLowTemperData = low;
        mHighTemperData = high;
        invalidate();
    }

    /**
     * 设置15天里面的最低和最高的温度数据
     *
     * @param low  最低温度
     * @param high 最高温度
     */
    public void setLowHighestData(int low, int high) {
        mLowestTemperData = low;
        mHighestTemperData = high;
        invalidate();
    }

    /**
     * 设置画笔信息
     */
    private void initPaint() {
        mTextPaint = new TextPaint(Paint.ANTI_ALIAS_FLAG);
        mTextPaint.setTextSize(mTemperTextSize);
        mTextPaint.setColor(mWeaTextColor);
        mTextFontMetrics = mTextPaint.getFontMetrics();

        mDotPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mDotPaint.setStyle(Paint.Style.FILL);
        mDotPaint.setColor(mWeaTextColor);

        mLinePaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mLinePaint.setStyle(Paint.Style.STROKE);
        mLinePaint.setStrokeWidth(mWeaLineWidth);
        mLinePaint.setColor(mWeaTextColor);
    }

    /**
     * 获取自定义属性并赋初始值
     */
    private void init(Context context, AttributeSet attrs, int defStyleAttr) {
        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.WeatherLineView,
                defStyleAttr, 0);
        mTemperTextSize = (int) a.getDimension(R.styleable.WeatherLineView_temperTextSize,
                dp2px(context, mTemperTextSize));
        mWeaTextColor = a.getColor(R.styleable.WeatherLineView_weatextColor, Color.parseColor("#b07b5c"));
        mWeaLineWidth = (int) a.getDimension(R.styleable.WeatherLineView_weaLineWidth,
                dp2px(context, mWeaLineWidth));
        mWeaDotRadius = (int) a.getDimension(R.styleable.WeatherLineView_weadotRadius,
                dp2px(context, mWeaDotRadius));
        mTextDotDistance = (int) a.getDimension(R.styleable.WeatherLineView_textDotDistance,
                dp2px(context, mTextDotDistance));
        a.recycle();
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        int width = getSize(widthMode, widthSize, 0);
        int height = getSize(heightMode, heightSize, 1);
        setMeasuredDimension(width, height);
    }

    /**
     * @param mode Mode
     * @param size Size
     * @param type 0表示宽度，1表示高度
     * @return 宽度或者高度
     */
    private int getSize(int mode, int size, int type) {
        // 默认
        int result;
        if (mode == MeasureSpec.EXACTLY) {
            result = size;
        } else {
            if (type == 0) {
                // 最小不能低于最小的宽度
                result = dp2px(getContext(), defaultMinWidth) + getPaddingLeft() + getPaddingRight();
            } else {
                // 最小不能小于最小的宽度加上一些数据
                int textHeight = (int) (mTextFontMetrics.bottom - mTextFontMetrics.top);
                // 加上2个文字的高度
                result = dp2px(getContext(), defaultMinHeight) + 2 * textHeight +
                        // 需要加上两个文字和圆点的间距
                        getPaddingTop() + getPaddingBottom() + 2 * mTextDotDistance;
            }

            if (mode == MeasureSpec.AT_MOST) {
                result = Math.min(result, size);
            }
        }
        return result;
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        if (mLowTemperData == null || mHighTemperData == null
                || mLowestTemperData == 0 || mHighestTemperData == 0) {
            return;
        }

        canvas.drawColor(Color.YELLOW);

        // 文本的高度
        int textHeight = (int) (mTextFontMetrics.bottom - mTextFontMetrics.top);

        // 一个基本的高度，由于最下面的时候，有文字和圆点和文字的宽度需要留空间
        int baseHeight = getHeight() - textHeight - mTextDotDistance;

        // 最低温度相关
        // 最低温度中间
        int calowMiddle = baseHeight - cacHeight(mLowTemperData[1]);
        canvas.drawCircle(getWidth() / 2, calowMiddle, mWeaDotRadius, mDotPaint);

        // 画温度文字
        String text = String.valueOf(mLowTemperData[1]) + "°";
        int baseX = (int) (canvas.getWidth() / 2 - mTextPaint.measureText(text) / 2);
        // mTextFontMetrics.top为负的
        // 需要加上文字高度和文字与圆点之间的空隙
        int baseY = (int) (calowMiddle - mTextFontMetrics.top) + mTextDotDistance;
        canvas.drawText(text, baseX, baseY, mTextPaint);

        if (mLowTemperData[0] != 0) {
            // 最低温度左边
            int calowLeft = baseHeight - cacHeight(mLowTemperData[0]);
            canvas.drawLine(0, calowLeft, getWidth() / 2, calowMiddle, mLinePaint);
        }

        if (mLowTemperData[2] != 0) {
            // 最低温度右边
            int calowRight = baseHeight - cacHeight(mLowTemperData[2]);
            canvas.drawLine(getWidth() / 2, calowMiddle, getWidth(), calowRight, mLinePaint);
        }

        // 最高温度相关
        // 最高温度中间
        int calHighMiddle = baseHeight - cacHeight(mHighTemperData[1]);
        canvas.drawCircle(getWidth() / 2, calHighMiddle, mWeaDotRadius, mDotPaint);

        // 画温度文字
        String text2 = String.valueOf(mHighTemperData[1]) + "°";
        int baseX2 = (int) (canvas.getWidth() / 2 - mTextPaint.measureText(text2) / 2);
        int baseY2 = (int) (calHighMiddle - mTextFontMetrics.bottom) - mTextDotDistance;
        canvas.drawText(text2, baseX2, baseY2, mTextPaint);

        if (mHighTemperData[0] != 0) {
            // 最高温度左边
            int calHighLeft = baseHeight - cacHeight(mHighTemperData[0]);
            canvas.drawLine(0, calHighLeft, getWidth() / 2, calHighMiddle, mLinePaint);
        }

        if (mHighTemperData[2] != 0) {
            // 最高温度右边
            int calHighRight = baseHeight - cacHeight(mHighTemperData[2]);
            canvas.drawLine(getWidth() / 2, calHighMiddle, getWidth(), calHighRight, mLinePaint);
        }
    }

    private int cacHeight(int tem) {
        // 最低，最高温度之差
        int temDistance = mHighestTemperData - mLowestTemperData;
        int textHeight = (int) (mTextFontMetrics.bottom - mTextFontMetrics.top);
        // view的最高和最低之差，需要减去文字高度和文字与圆点之间的空隙
        int viewDistance = getHeight() - 2 * textHeight - 2 * mTextDotDistance;
        // 今天的温度和最低温度之间的差别
        int currTemDistance = tem - mLowestTemperData;
        return currTemDistance * viewDistance / temDistance;
    }

    public static int dp2px(Context context, float dpVal) {
        return (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP,
                dpVal, context.getResources().getDisplayMetrics());
    }
}
```
上面就是折线的天气View，代码的注释比较详细了。


#### RecyclerView的Item的布局
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:orientation="horizontal">

    <View
        android:layout_width="0.5dp"
        android:layout_height="match_parent"
        android:background="#A04D4E"/>

    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <View
            android:layout_width="match_parent"
            android:layout_height="1dp"
            android:background="#A04D4E"/>

        <TextView
            android:id="@+id/id_day_text_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:layout_marginTop="10dp"
            android:text="多云"
            android:textSize="18sp"/>

        <ImageView
            android:id="@+id/id_day_icon_iv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:layout_marginTop="10dp"
            android:src="@drawable/wth_code_99"/>

        <yong.qing.com.qimingview.weatherview.WeatherLineView
            android:id="@+id/wea_line"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>

        <ImageView
            android:id="@+id/id_night_icon_iv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:src="@drawable/wth_code_99"/>

        <TextView
            android:id="@+id/id_night_text_tv"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:layout_marginBottom="10dp"
            android:layout_marginTop="10dp"
            android:text="多云"
            android:textSize="18sp"/>

        <View
            android:layout_width="match_parent"
            android:layout_height="1dp"
            android:background="#A04D4E"/>

    </LinearLayout>

    <View
        android:layout_width="0.5dp"
        android:layout_height="match_parent"
        android:background="#A04D4E"/>
</LinearLayout>
```
预览的效果是这样的:
![预览](/images/weather_impl_item.png)
当然这里那些属性我没有加上去，因为代码里面有设置默认值，如果觉得不满足要求的话，可以自己设置。

#### RecyclerView的adapter的实现
```java
public class WeaDataAdapter extends RecyclerView.Adapter<WeaDataAdapter.WeatherDataViewHolder> {

    private Context mContext;
    private LayoutInflater mInflater;
    private List<WeatherDailyModel> mDatas;
    private int mLowestTem;
    private int mHighestTem;

    public WeaDataAdapter(Context context, List<WeatherDailyModel> datats, int lowtem, int hightem) {
        mContext = context;
        mInflater = LayoutInflater.from(context);
        mDatas = datats;
        mLowestTem = lowtem;
        mHighestTem = hightem;
    }

    @Override
    public WeatherDataViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view = mInflater.inflate(R.layout.item_weather_item, parent, false);
        WeatherDataViewHolder viewHolder = new WeatherDataViewHolder(view);
        viewHolder.dayText = (TextView) view.findViewById(R.id.id_day_text_tv);
        viewHolder.dayIcon = (ImageView) view.findViewById(R.id.id_day_icon_iv);
        viewHolder.weatherLineView = (WeatherLineView) view.findViewById(R.id.wea_line);
        viewHolder.nighticon = (ImageView) view.findViewById(R.id.id_night_icon_iv);
        viewHolder.nightText = (TextView) view.findViewById(R.id.id_night_text_tv);
        return viewHolder;
    }

    @Override
    public void onBindViewHolder(WeatherDataViewHolder holder, int position) {
        // 最低温度设置为15，最高温度设置为30
        Resources resources = mContext.getResources();
        WeatherDailyModel weatherModel = mDatas.get(position);
        holder.dayText.setText(weatherModel.getText_day());
        int iconday = resources.getIdentifier("wth_code_" + weatherModel.getCode_day(), "drawable", mContext.getPackageName());
        if (iconday == 0) {
            holder.dayIcon.setImageResource(R.drawable.wth_code_99);
        } else {
            holder.dayIcon.setImageResource(iconday);
        }
        holder.weatherLineView.setLowHighestData(mLowestTem, mHighestTem);
        int iconight = resources.getIdentifier("wth_code_" + weatherModel.getCode_day(), "drawable", mContext.getPackageName());
        if (iconight == 0) {
            holder.nighticon.setImageResource(R.drawable.wth_code_99);
        } else {
            holder.nighticon.setImageResource(iconight);
        }
        holder.nightText.setText(weatherModel.getText_night());
        int low[] = new int[3];
        int high[] = new int[3];
        low[1] = weatherModel.getLow();
        high[1] = weatherModel.getHigh();
        if (position <= 0) {
            low[0] = 0;
            high[0] = 0;
        } else {
            WeatherDailyModel weatherModelLeft = mDatas.get(position - 1);
            low[0] = (weatherModelLeft.getLow() + weatherModel.getLow()) / 2;
            high[0] = (weatherModelLeft.getHigh() + weatherModel.getHigh()) / 2;
        }
        if (position >= mDatas.size() - 1) {
            low[2] = 0;
            high[2] = 0;
        } else {
            WeatherDailyModel weatherModelRight = mDatas.get(position + 1);
            low[2] = (weatherModel.getLow() + weatherModelRight.getLow()) / 2;
            high[2] = (weatherModel.getHigh() + weatherModelRight.getHigh()) / 2;
        }
        holder.weatherLineView.setLowHighData(low, high);
    }

    @Override
    public int getItemCount() {
        return mDatas.size();
    }

    public static class WeatherDataViewHolder extends RecyclerView.ViewHolder {

        TextView dayText;
        ImageView dayIcon;
        WeatherLineView weatherLineView;
        ImageView nighticon;
        TextView nightText;

        public WeatherDataViewHolder(View itemView) {
            super(itemView);
        }
    }
}
```
Model里面的字段:
```java
public static class WeatherDailyModel {
    /**
        * date : 2016-05-30
        * text_day : 多云
        * code_day : 4
        * text_night : 阴
        * code_night : 9
        * high : 34
        * low : 22
        */
    private String date;
    private String text_day;
    private int code_day;
    private String text_night;
    private int code_night;
    private int high;
    private int low;
}
```

#### Activity里面获取数组设置到RecyclerView里面去
```java
private void initView() {
    //得到控件
    mRecyclerView = (RecyclerView) findViewById(R.id.id_recyclerview_horizontal);
    //设置布局管理器
    LinearLayoutManager layoutManager = new LinearLayoutManager(this);
    layoutManager.setOrientation(LinearLayoutManager.HORIZONTAL);
    mRecyclerView.setLayoutManager(layoutManager);
}
```

```java
private void fillDatatoRecyclerView(List<WeatherDailyModel> daily) {
    mWeatherModels = daily;
    Collections.sort(daily, new Comparator<WeatherDailyModel>() {
        @Override
        public int compare(WeatherDailyModel lhs,
                            WeatherDailyModel rhs) {
            // 排序找到温度最低的，按照最低温度升序排列
            return lhs.getLow() - rhs.getLow();
        }
    });

    int low = daily.get(0).getLow();

    Collections.sort(daily, new Comparator<WeatherDailyModel>() {
        @Override
        public int compare(WeatherDailyModel lhs,
                            WeatherDailyModel rhs) {
            // 排序找到温度最高的，按照最高温度降序排列
            return rhs.getHigh() - lhs.getHigh();
        }
    });
    int high = daily.get(0).getHigh();

    mWeaDataAdapter = new WeaDataAdapter(this, mWeatherModels, low, high);
    mRecyclerView.setAdapter(mWeaDataAdapter);
}
```
这样其实就搞定了，WeatherLineView可能比较麻烦一点吧，但是只要是想清楚了，就很好了，从上面也看到WeatherLineView没什么含金量的，就是很普通的绘制。