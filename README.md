# 自定义彩色进度条效果 #

## 效果如下：颜色可以随意设置 ##

![](https://i.imgur.com/GIUACaI.gif)

## View代码 ##

	  1 package angus.wsj.com.circularprogressview;
	  2 
	  3 import android.content.Context;
	  4 import android.util.AttributeSet;
	  5 import android.view.View;
	  6 import android.animation.Animator;
	  7 import android.animation.AnimatorListenerAdapter;
	  8 import android.animation.AnimatorSet;
	  9 import android.animation.ValueAnimator;
	 10 import android.content.res.Resources;
	 11 import android.content.res.TypedArray;
	 12 import android.graphics.Canvas;
	 13 import android.graphics.Color;
	 14 import android.graphics.LinearGradient;
	 15 import android.graphics.Paint;
	 16 import android.graphics.RectF;
	 17 import android.graphics.Shader;
	 18 import android.os.Build;
	 19 import android.util.TypedValue;
	 20 import android.view.animation.DecelerateInterpolator;
	 21 import android.view.animation.LinearInterpolator;
	 22 
	 23 /**
	 24  * Created by Angus
	 25  * Angus on 2018/1/20 20:46
	 26  */
	 27 
	 28 public class CircularProgressView extends View {
	 29 
	 30     private static final float INDETERMINANT_MIN_SWEEP = 15f;
	 31 
	 32     private Paint paint;
	 33     private int size = 0;
	 34     private RectF bounds;
	 35 
	 36     private boolean isIndeterminate, autostartAnimation;
	 37     private float currentProgress, maxProgress, indeterminateSweep, indeterminateRotateOffset;
	 38     private int thickness, color, animDuration, animSwoopDuration, animSyncDuration, animSteps;
	 39 
	 40 
	 41     // Animation related stuff
	 42     private float startAngle;
	 43     private float actualProgress;
	 44     private ValueAnimator startAngleRotate;
	 45     private ValueAnimator progressAnimator;
	 46     private AnimatorSet indeterminateAnimator;
	 47     private float initialStartAngle;
	 48     private int[] mColors;//渐变色数组
	 49 
	 50     public CircularProgressView(Context context) {
	 51         super(context);
	 52         init(null, 0);
	 53     }
	 54 
	 55     public CircularProgressView(Context context, AttributeSet attrs) {
	 56         super(context, attrs);
	 57         init(attrs, 0);
	 58     }
	 59 
	 60     public CircularProgressView(Context context, AttributeSet attrs, int defStyle) {
	 61         super(context, attrs, defStyle);
	 62         init(attrs, defStyle);
	 63     }
	 64 
	 65     protected void init(AttributeSet attrs, int defStyle) {
	 66 
	 67         //设置渐变颜色
	 68         Shader s = new LinearGradient(0, 0, 100, 100,
	 69                 new int[] { Color.RED, Color.GREEN, Color.BLUE, Color.YELLOW,
	 70                         Color.LTGRAY }, null, Shader.TileMode.REPEAT);
	 71         initAttributes(attrs, defStyle);
	 72 
	 73         paint = new Paint(Paint.ANTI_ALIAS_FLAG);
	 74         paint.setStyle(Paint.Style.STROKE);
	 75         paint.setStrokeWidth(thickness);
	 76         paint.setStrokeCap(Paint.Cap.BUTT);
	 77         paint.setShader(s);
	 78 
	 79         bounds = new RectF();
	 80     }
	 81 
	 82     private void initAttributes(AttributeSet attrs, int defStyle)
	 83     {
	 84         final TypedArray a = getContext().obtainStyledAttributes(
	 85                 attrs, R.styleable.CircularProgressView, defStyle, 0);
	 86 
	 87         Resources resources = getResources();
	 88 
	 89         // 初始化属性
	 90         currentProgress = a.getFloat(R.styleable.CircularProgressView_cpv_progress,
	 91                 resources.getInteger(R.integer.cpv_default_progress));
	 92         maxProgress = a.getFloat(R.styleable.CircularProgressView_cpv_maxProgress,
	 93                 resources.getInteger(R.integer.cpv_default_max_progress));
	 94         thickness = a.getDimensionPixelSize(R.styleable.CircularProgressView_cpv_thickness,
	 95                 resources.getDimensionPixelSize(R.dimen.cpv_default_thickness));
	 96         isIndeterminate = a.getBoolean(R.styleable.CircularProgressView_cpv_indeterminate,
	 97                 resources.getBoolean(R.bool.cpv_default_is_indeterminate));
	 98         autostartAnimation = a.getBoolean(R.styleable.CircularProgressView_cpv_animAutostart,
	 99                 resources.getBoolean(R.bool.cpv_default_anim_autostart));
	100         initialStartAngle = a.getFloat(R.styleable.CircularProgressView_cpv_startAngle,
	101                 resources.getInteger(R.integer.cpv_default_start_angle));
	102         startAngle = initialStartAngle;
	103 
	104         int accentColor = getContext().getResources().getIdentifier("colorAccent", "attr", getContext().getPackageName());
	105 
	106         // If color explicitly provided
	107         if (a.hasValue(R.styleable.CircularProgressView_cpv_color)) {
	108             color = a.getColor(R.styleable.CircularProgressView_cpv_color, resources.getColor(R.color.cpv_default_color));
	109         }
	110         // If using support library v7 accentColor
	111         else if(accentColor != 0) {
	112             TypedValue t = new TypedValue();
	113             getContext().getTheme().resolveAttribute(accentColor, t, true);
	114             color = t.data;
	115         }
	116         // If using native accentColor (SDK >21)
	117         else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
	118             TypedArray t = getContext().obtainStyledAttributes(new int[] { android.R.attr.colorAccent });
	119             color = t.getColor(0, resources.getColor(R.color.cpv_default_color));
	120         }
	121         else {
	122             //Use default color
	123             color = resources.getColor(R.color.cpv_default_color);
	124         }
	125 
	126         animDuration = a.getInteger(R.styleable.CircularProgressView_cpv_animDuration,
	127                 resources.getInteger(R.integer.cpv_default_anim_duration));
	128         animSwoopDuration = a.getInteger(R.styleable.CircularProgressView_cpv_animSwoopDuration,
	129                 resources.getInteger(R.integer.cpv_default_anim_swoop_duration));
	130         animSyncDuration = a.getInteger(R.styleable.CircularProgressView_cpv_animSyncDuration,
	131                 resources.getInteger(R.integer.cpv_default_anim_sync_duration));
	132         animSteps = a.getInteger(R.styleable.CircularProgressView_cpv_animSteps,
	133                 resources.getInteger(R.integer.cpv_default_anim_steps));
	134         a.recycle();
	135     }
	136 
	137     @Override
	138     protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
	139         super.onMeasure(widthMeasureSpec, heightMeasureSpec);
	140         int xPad = getPaddingLeft() + getPaddingRight();
	141         int yPad = getPaddingTop() + getPaddingBottom();
	142         int width = getMeasuredWidth() - xPad;
	143         int height = getMeasuredHeight() - yPad;
	144         size = (width < height) ? width : height;
	145         setMeasuredDimension(size + xPad, size + yPad);
	146 
	147     }
	148 
	149     @Override
	150     protected void onSizeChanged(int w, int h, int oldw, int oldh) {
	151         super.onSizeChanged(w, h, oldw, oldh);
	152         size = (w < h) ? w : h;
	153         updateBounds();
	154     }
	155 
	156     private void updateBounds()
	157     {
	158         int paddingLeft = getPaddingLeft();
	159         int paddingTop = getPaddingTop();
	160         bounds.set(paddingLeft + thickness, paddingTop + thickness, size - paddingLeft - thickness, size - paddingTop - thickness);
	161     }
	162 
	163 
	164 
	165     @Override
	166     protected void onDraw(Canvas canvas) {
	167         super.onDraw(canvas);
	168 
	169         // Draw the arc
	170         float sweepAngle = (isInEditMode()) ? currentProgress/maxProgress*360 : actualProgress/maxProgress*360;
	171         if(!isIndeterminate)
	172             canvas.drawArc(bounds, startAngle, sweepAngle, false, paint);
	173         else
	174             canvas.drawArc(bounds, startAngle + indeterminateRotateOffset, indeterminateSweep, false, paint);
	175     }
	176 
	177 
	178 
	179     /**
	180      * Get the thickness of the progress bar arc.
	181      * @return the thickness of the progress bar arc
	182      */
	183     public int getThickness() {
	184         return thickness;
	185     }
	186 
	187     /**
	188      * Sets the thickness of the progress bar arc.
	189      * @param thickness the thickness of the progress bar arc
	190      */
	191     public void setThickness(int thickness) {
	192         this.thickness = thickness;
	193         updateBounds();
	194         invalidate();
	195     }
	196 
	197     /**
	198      * Gets the progress value considered to be 100% of the progress bar.
	199      * @return the maximum progress
	200      */
	201     public float getMaxProgress() {
	202         return maxProgress;
	203     }
	204 
	205     /**
	206      * Sets the progress value considered to be 100% of the progress bar.
	207      * @param maxProgress the maximum progress
	208      */
	209     public void setMaxProgress(float maxProgress) {
	210         this.maxProgress = maxProgress;
	211         invalidate();
	212     }
	213 
	214     /**
	215      * @return current progress
	216      */
	217     public float getProgress() {
	218         return currentProgress;
	219     }
	220 
	221     /**
	222      * Sets the progress of the progress bar.
	223      *
	224      * @param currentProgress the new progress.
	225      */
	226     public void setProgress(final float currentProgress) {
	227         this.currentProgress = currentProgress;
	228         // Reset the determinate animation to approach the new currentProgress
	229         if (!isIndeterminate) {
	230             if (progressAnimator != null && progressAnimator.isRunning())
	231                 progressAnimator.cancel();
	232             progressAnimator = ValueAnimator.ofFloat(actualProgress, currentProgress);
	233             progressAnimator.setDuration(animSyncDuration);
	234             progressAnimator.setInterpolator(new LinearInterpolator());
	235             progressAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	236                 @Override
	237                 public void onAnimationUpdate(ValueAnimator animation) {
	238                     actualProgress = (Float) animation.getAnimatedValue();
	239                     invalidate();
	240                 }
	241             });
	242 
	243 
	244             progressAnimator.start();
	245         }
	246         invalidate();
	247 
	248     }
	249 
	250 
	251 
	252     /**
	253      * Starts the progress bar animation.
	254      * (This is an alias of resetAnimation() so it does the same thing.)
	255      */
	256     public void startAnimation() {
	257         resetAnimation();
	258     }
	259 
	260     /**
	261      * Resets the animation.
	262      */
	263     public void resetAnimation() {
	264         // Cancel all the old animators
	265         if(startAngleRotate != null && startAngleRotate.isRunning())
	266             startAngleRotate.cancel();
	267         if(progressAnimator != null && progressAnimator.isRunning())
	268             progressAnimator.cancel();
	269         if(indeterminateAnimator != null && indeterminateAnimator.isRunning())
	270             indeterminateAnimator.cancel();
	271 
	272         // Determinate animation
	273         if(!isIndeterminate)
	274         {
	275             // The cool 360 swoop animation at the start of the animation
	276             startAngle = initialStartAngle;
	277             startAngleRotate = ValueAnimator.ofFloat(startAngle, startAngle + 360);
	278             startAngleRotate.setDuration(animSwoopDuration);
	279             startAngleRotate.setInterpolator(new DecelerateInterpolator(2));
	280             startAngleRotate.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	281                 @Override
	282                 public void onAnimationUpdate(ValueAnimator animation) {
	283                     startAngle = (Float) animation.getAnimatedValue();
	284                     invalidate();
	285                 }
	286             });
	287             startAngleRotate.start();
	288 
	289             // The linear animation shown when progress is updated
	290             actualProgress = 0f;
	291             progressAnimator = ValueAnimator.ofFloat(actualProgress, currentProgress);
	292             progressAnimator.setDuration(animSyncDuration);
	293             progressAnimator.setInterpolator(new LinearInterpolator());
	294             progressAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	295                 @Override
	296                 public void onAnimationUpdate(ValueAnimator animation) {
	297                     actualProgress = (Float) animation.getAnimatedValue();
	298                     invalidate();
	299                 }
	300             });
	301             progressAnimator.start();
	302         }
	303         // Indeterminate animation
	304         else
	305         {
	306             indeterminateSweep = INDETERMINANT_MIN_SWEEP;
	307             // Build the whole AnimatorSet
	308             indeterminateAnimator = new AnimatorSet();
	309             AnimatorSet prevSet = null, nextSet;
	310             for(int k=0;k<animSteps;k++)
	311             {
	312                 nextSet = createIndeterminateAnimator(k);
	313                 AnimatorSet.Builder builder = indeterminateAnimator.play(nextSet);
	314                 if(prevSet != null)
	315                     builder.after(prevSet);
	316                 prevSet = nextSet;
	317             }
	318 
	319             // Listen to end of animation so we can infinitely loop
	320             indeterminateAnimator.addListener(new AnimatorListenerAdapter() {
	321                 boolean wasCancelled = false;
	322                 @Override
	323                 public void onAnimationCancel(Animator animation) {
	324                     wasCancelled = true;
	325                 }
	326 
	327                 @Override
	328                 public void onAnimationEnd(Animator animation) {
	329                     if(!wasCancelled)
	330                         resetAnimation();
	331                 }
	332             });
	333             indeterminateAnimator.start();
	334 
	335         }
	336 
	337 
	338     }
	339 
	340     /**
	341      * Stops the animation
	342      */
	343 
	344     public void stopAnimation() {
	345         if(startAngleRotate != null) {
	346             startAngleRotate.cancel();
	347             startAngleRotate = null;
	348         }
	349         if(progressAnimator != null) {
	350             progressAnimator.cancel();
	351             progressAnimator = null;
	352         }
	353         if(indeterminateAnimator != null) {
	354             indeterminateAnimator.cancel();
	355             indeterminateAnimator = null;
	356         }
	357     }
	358 
	359     // Creates the animators for one step of the animation
	360     private AnimatorSet createIndeterminateAnimator(float step)
	361     {
	362         final float maxSweep = 360f*(animSteps-1)/animSteps + INDETERMINANT_MIN_SWEEP;
	363         final float start = -90f + step*(maxSweep-INDETERMINANT_MIN_SWEEP);
	364 
	365         // Extending the front of the arc
	366         ValueAnimator frontEndExtend = ValueAnimator.ofFloat(INDETERMINANT_MIN_SWEEP, maxSweep);
	367         frontEndExtend.setDuration(animDuration/animSteps/2);
	368         frontEndExtend.setInterpolator(new DecelerateInterpolator(1));
	369         frontEndExtend.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	370             @Override
	371             public void onAnimationUpdate(ValueAnimator animation) {
	372                 indeterminateSweep = (Float) animation.getAnimatedValue();
	373                 invalidate();
	374             }
	375         });
	376 
	377         // Overall rotation
	378         ValueAnimator rotateAnimator1 = ValueAnimator.ofFloat(step*720f/animSteps, (step+.5f)*720f/animSteps);
	379         rotateAnimator1.setDuration(animDuration/animSteps/2);
	380         rotateAnimator1.setInterpolator(new LinearInterpolator());
	381         rotateAnimator1.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	382             @Override
	383             public void onAnimationUpdate(ValueAnimator animation) {
	384                 indeterminateRotateOffset = (Float) animation.getAnimatedValue();
	385             }
	386         });
	387 
	388         // Followed by...
	389 
	390         // Retracting the back end of the arc
	391         ValueAnimator backEndRetract = ValueAnimator.ofFloat(start, start+maxSweep-INDETERMINANT_MIN_SWEEP);
	392         backEndRetract.setDuration(animDuration/animSteps/2);
	393         backEndRetract.setInterpolator(new DecelerateInterpolator(1));
	394         backEndRetract.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	395             @Override
	396             public void onAnimationUpdate(ValueAnimator animation) {
	397                 startAngle = (Float) animation.getAnimatedValue();
	398                 indeterminateSweep = maxSweep - startAngle + start;
	399                 invalidate();
	400             }
	401         });
	402 
	403         // More overall rotation
	404         ValueAnimator rotateAnimator2 = ValueAnimator.ofFloat((step + .5f) * 720f / animSteps, (step + 1) * 720f / animSteps);
	405         rotateAnimator2.setDuration(animDuration / animSteps / 2);
	406         rotateAnimator2.setInterpolator(new LinearInterpolator());
	407         rotateAnimator2.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
	408             @Override
	409             public void onAnimationUpdate(ValueAnimator animation) {
	410                 indeterminateRotateOffset = (Float) animation.getAnimatedValue();
	411             }
	412         });
	413 
	414         AnimatorSet set = new AnimatorSet();
	415         set.play(frontEndExtend).with(rotateAnimator1);
	416         set.play(backEndRetract).with(rotateAnimator2).after(rotateAnimator1);
	417         return set;
	418     }
	419 
	420     @Override
	421     protected void onAttachedToWindow() {
	422         super.onAttachedToWindow();
	423         if(autostartAnimation)
	424             startAnimation();
	425     }
	426 
	427     @Override
	428     protected void onDetachedFromWindow() {
	429         super.onDetachedFromWindow();
	430         stopAnimation();
	431     }
	432 
	433     @Override
	434     public void setVisibility(int visibility) {
	435         int currentVisibility = getVisibility();
	436         super.setVisibility(visibility);
	437         if (visibility != currentVisibility) {
	438             if (visibility == View.VISIBLE){
	439                 resetAnimation();
	440             } else if (visibility == View.GONE || visibility == View.INVISIBLE) {
	441                 stopAnimation();
	442             }
	443         }
	444     }
	445 }







## MainActivity代码设置了下延时效果： ##

	  1 package angus.wsj.com.circularprogressview;
	  2 
	  3 import android.os.Handler;
	  4 import android.os.SystemClock;
	  5 import android.support.v7.app.AppCompatActivity;
	  6 import android.os.Bundle;
	  7 
	  8 /**
	  9  * Created by Angus
	 10  * Angus on 2018/1/20 20:46
	 11  * 自定义彩色进度条
	 12  */
	 13 public class MainActivity extends AppCompatActivity {
	 14     CircularProgressView progressView;
	 15     Thread updateThread;
	 16     @Override
	 17     protected void onCreate(Bundle savedInstanceState) {
	 18         super.onCreate(savedInstanceState);
	 19         setContentView(R.layout.activity_main);
	 20         progressView = findViewById(R.id.progress_view);
	 21         startAnimationThreadStuff(1000);
	 22     }
	 23     private void startAnimationThreadStuff(long delay) {
	 24         if (updateThread != null && updateThread.isAlive())
	 25             updateThread.interrupt();
	 26         // 开启延迟动画
	 27         final Handler handler = new Handler();
	 28         handler.postDelayed(new Runnable() {
	 29             @Override
	 30             public void run() {
	 31 
	 32                     progressView.setProgress(0f);
	 33 
	 34                     updateThread = new Thread(new Runnable() {
	 35                         @Override
	 36                         public void run() {
	 37                             while (progressView.getProgress() < progressView.getMaxProgress() && !Thread.interrupted()) {
	 38 
	 39                                 handler.post(new Runnable() {
	 40                                     @Override
	 41                                     public void run() {
	 42                                         progressView.setProgress(progressView.getProgress() + 10);
	 43                                     }
	 44                                 });
	 45                                 SystemClock.sleep(250);
	 46                             }
	 47                         }
	 48                     });
	 49                     updateThread.start();
	 50 
	 51 
	 52                 progressView.startAnimation();
	 53             }
	 54         }, delay);
	 55     }
	 56 }

