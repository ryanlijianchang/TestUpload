## 功能需求 ##

最近自己在开发一个社交APP，发送动态（类似朋友圈）是社交APP必备的一个功能，而自己在开发过程中也需要开发到这一个功能，但是在开发中遇到了一个问题，就是如何绘制一个类似朋友群那样动态添加图片，并加号随着自己的图片增加而后移这一个UI，而这篇小文就是教你如何制作一个仿朋友圈发带图朋友圈的UI设计。注意，这是UI设计，并不是实现图片上传功能。

当然，如果你想知道如何实现图片上传到服务器，请看我的另一篇文章：[Android学习-使用Async-Http实现图片压缩并上传功能](http://blog.csdn.net/ljcitworld/article/details/51670910)。个人水平有限，如有不足的地方，欢迎交流，勿喷。

## 效果图 ##

按照惯例，先上效果图。

![](http://7xrwkh.com1.z0.glb.clouddn.com/Post-SendPhotoUI2.png)

## 困难 ##

在自己开发学习过程中，主要遇到了两个难点：

1. 添加过多图片时，会出现OOM。
2. 如何动态修改图片展示栏的高度。
3. 加号如何伴随图片的增加而后移。
4. 如何保证最多添加照片为9张。


## 难点解决 ##

### 添加过多图片时，会出现OOM ###

出现第一种情况的原因很简单，就是随着我们手机的像素越来越高，图片的大小也越来越大，我们普通的机拍出来照片至少也有1~2M，更不说像素高的手机。而对于一个安卓应用来说，由于手机设备的限制，一般应用使用的RAM不能超过某个设定值，不同产商默认值不太一样，一般常见的有16M，24M，32M,48M。所以一个Activity中加载几张高清原图，就会报Out Of Memory 错误，也就是所谓的OOM错误。所以知道了这个问题之后我们就很容易解决了，我们就可以先将图片压缩，然后再使用ImageView加载压缩后的图片即可。而我们这里是通过对图片的尺寸进行压缩实现图片的压缩，这里大概说一下。


1. 要对图片压缩，首先要先将BitmapFactory.Options中的inJustDecodeBounds设置为true。

    	final BitmapFactory.Options options = new BitmapFactory.Options();
		// 若要对图片进行压缩，必须先设置Option的inJustDecodeBounds为true
		options.inJustDecodeBounds = true;

2. 然后通过BitmapFactory中decodeFile方法来获取到照片的高度和宽度，这里只要存进一个图片地址即可。获取图片地址这里就不详讲了。

		BitmapFactory.decodeFile(pathName，options)




3. 然后需要对BitmapFactory.Options中的inSampleSize根据你需要压缩比例进行设置，options.inSampleSize是图片的压缩比，例如原来大小是100 * 100，options.inSampleSize为1，则不变，options.inSampleSize为2，则压缩成50 * 50。而我这里是根据自己设置最低宽度和最低高度来获取inSampleSize的值：

	    private static int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHeight) {
			final int height = options.outHeight;
			final int width = options.outWidth;
			int inSampleSize = 1;
			if (height > reqHeight || width > reqWidth) {
				//首先获取原图高度和宽度的一半
				final int halfHeight = height / 2;
				final int halfWidth = width / 2;
				//循环，如果halfHeight和halfWidth同时大于最小宽度和最小高度时，inSampleSize乘2，
				//最后得到的宽或者高都是最接近最小宽度或者最小高度的
				while ((halfHeight / inSampleSize) > reqHeight && (halfWidth / inSampleSize) > reqWidth) {
					inSampleSize *= 2;
				}
			}
			return inSampleSize;
		}

4. 获取到inSampleSize值之后，重新设置options.inJustDecodeBounds为false，不能修改option，调用BitmapFactory中的decodeFile方法即可获取到压缩后的照片，这样在加载图片时就可以避免OOM的出现了。

		options.inJustDecodeBounds = false;
		// 根据options重新加载图片
		Bitmap src = BitmapFactory.decodeFile(pathName, options);

综上，我将按尺寸压缩照片的功能包装成BitmapUtil类，在使用时直接调用即可。

    public class BitmapUtils {

		private static int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHeight) {
			final int height = options.outHeight;
			final int width = options.outWidth;
			int inSampleSize = 1;
			if (height > reqHeight || width > reqWidth) {
				final int halfHeight = height / 2;
				final int halfWidth = width / 2;
				while ((halfHeight / inSampleSize) > reqHeight && (halfWidth / inSampleSize) > reqWidth) {
					inSampleSize *= 2;
				}
			}
			return inSampleSize;
		}
	
		/**
		 * 根据Resources压缩图片
		 * 
		 * @param res
		 * @param resId
		 * @param reqWidth
		 * @param reqHeight
		 * @return
		 */
		public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId, int reqWidth, int reqHeight) {
			final BitmapFactory.Options options = new BitmapFactory.Options();
			options.inJustDecodeBounds = true;
			BitmapFactory.decodeResource(res, resId, options);
			options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
			options.inJustDecodeBounds = false;
			Bitmap src = BitmapFactory.decodeResource(res, resId, options);
			return src;
		}
	
		/**
		 * 根据地址压缩图片
		 * 
		 * @param pathName
		 * @param reqWidth
		 * @param reqHeight
		 * @return
		 */
		public static Bitmap decodeSampledBitmapFromFd(String pathName, int reqWidth, int reqHeight) {
			final BitmapFactory.Options options = new BitmapFactory.Options();
			options.inJustDecodeBounds = true;
			BitmapFactory.decodeFile(pathName, options);
			options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);
			options.inJustDecodeBounds = false;
			Bitmap src = BitmapFactory.decodeFile(pathName, options);
			return src;
		}
	}




### 如何动态修改图片展示栏的高度 ###

如何动态修改图片展示栏的高度，首先我说一下我是使用GridView来实现图片栏的展示，所以我们可以在第一次加载GridView时可以获取到下图的参数，大家看图会容易理解一点。


- 我们的照片如果只有一栏，则GridView的高度不变
- 如果照片有两栏，则高度设置为gridViewH * 2 - (gridViewH - imageViewH) / 2
- 如果有三栏，则GrideView的高度设置为gridViewH * 3 - (gridViewH - imageViewH)

![](http://7xrwkh.com1.z0.glb.clouddn.com/Post-SendPhotoUI3.png)



1. 我们在第一次加载GridView时记录GridView的高度GridViewH。
	
		LinearLayout.LayoutParams params =(android.widget.LinearLayout.LayoutParams) mGridView.getLayoutParams();

		gridViewH = params.height;

2. 同时记录ImageView的高度

		RelativeLayout.LayoutParams params = (android.widget.RelativeLayout.LayoutParams) holder.imageView
					.getLayoutParams();
		imageViewH = params.height;

3. 则上下的边距为

		(gridViewH - imageViewH) / 2

4. 将它写成一个方法，在每次getView()方法中调用即可。

    	private void setGridView() {
			LinearLayout.LayoutParams lp = (android.widget.LinearLayout.LayoutParams) mGridView.getLayoutParams();
			if (data.size() < 4) {
				lp.height = gridViewH;
			} else if (data.size() < 8) {
				lp.height = gridViewH * 2 - (gridViewH - imageViewH) / 2;
			} else {
				lp.height = gridViewH * 3 - (gridViewH - imageViewH);
			}
			mGridView.setLayoutParams(lp);
		}


### 加号如何伴随图片的增加而后移 ###

因为我的数据源是List<Bitmap>，所以可以这么做：

- 当第一次加载时，List中只有一张加号的照片
- 当添加了照片之后，List先移除加号照片，再添加照片，最后再把加号照片添加进去
	
    		data.remove(data.size() - 1);
			Bitmap bp = BitmapFactory.decodeResource(getResources(), R.drawable.ic_addpic);
			data.add(newBp);
			data.add(bp);
			//将路径设置为空，防止在手机休眠后返回Activity调用此方法时添加照片
			photoPath = null;
			adapter.notifyDataSetChanged();

### 如何保证最多添加照片为9张 ###

这个问题只需要在每次添加之前判断数据源的大小是否为10（包括加号照片，大小就为10）。

	if (data.size() == 10) {
		Toast.makeText(MainActivity.this, "图片数9张已满", Toast.LENGTH_SHORT).show();
	} else {
		if (position == data.size() - 1) {
			Toast.makeText(MainActivity.this, "添加图片", Toast.LENGTH_SHORT).show();
			// 选择图片
			Intent intent = new Intent(Intent.ACTION_PICK, null);
			intent.setDataAndType(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, "image/*");
			startActivityForResult(intent, 0x1);
		} else {
			Toast.makeText(MainActivity.this, "点击第" + (position + 1) + " 号图片", Toast.LENGTH_SHORT).show();
		}
	}



## 界面 ##

activity_main.xml

    <ScrollView xmlns:android="http://schemas.android.com/apk/res/android"
	    xmlns:tools="http://schemas.android.com/tools"
	    android:layout_width="match_parent"
	    android:layout_height="match_parent"
	    android:background="#F3F6F8"
	    android:orientation="vertical" >
	
	    <LinearLayout
	        android:layout_width="match_parent"
	        android:layout_height="wrap_content"
	        android:orientation="vertical" >
	
	        <ImageView
	            android:layout_width="match_parent"
	            android:layout_height="1dp"
	            android:layout_marginTop="2dp"
	            android:src="#E4E3E3" />
	
	        <EditText
	            android:id="@+id/content_et"
	            android:layout_width="fill_parent"
	            android:layout_height="120dp"
	            android:background="#FFFFFF"
	            android:gravity="top"
	            android:hint="随手说出你此刻的心声..."
	            android:maxLength="500"
	            android:padding="5dp"
	            android:singleLine="false"
	            android:textColor="#000000"
	            android:textSize="20sp" />
	
	        <ImageView
	            android:layout_width="match_parent"
	            android:layout_height="1dp"
	            android:src="#E4E3E3" />
	
	        <ImageView
	            android:layout_width="match_parent"
	            android:layout_height="1dp"
	            android:layout_marginTop="10dp"
	            android:src="#E4E3E3" />
	
	        <GridView
	            android:id="@+id/gridView1"
	            android:layout_width="fill_parent"
	            android:layout_height="100dp"
	            android:background="#FFFFFF"
	            android:columnWidth="90dp"
	            android:gravity="center"
	            android:horizontalSpacing="5dp"
	            android:numColumns="4"
	            android:padding="10dp"
	            android:stretchMode="columnWidth"
	            android:verticalSpacing="5dp" >
	        </GridView>
	
	        <ImageView
	            android:layout_width="match_parent"
	            android:layout_height="1dp"
	            android:src="#E4E3E3" />
	
	        <Button
	            android:id="@+id/send_btn"
	            android:layout_width="match_parent"
	            android:layout_height="wrap_content"
	            android:layout_gravity="center"
	            android:layout_marginLeft="35dp"
	            android:layout_marginRight="35dp"
	            android:layout_marginTop="20dp"
	            android:background="@drawable/send_btn_selector"
	            android:gravity="center"
	            android:text="发送"
	            android:textColor="#FFFFFF"
	            android:textSize="20sp" />
	    </LinearLayout>

	</ScrollView>

griditem.xml

    <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
	    android:layout_width="80dp"
	    android:layout_height="80dp"
	    android:descendantFocusability="blocksDescendants"
	    android:gravity="center" >
	
	    <ImageView
	        android:id="@+id/imageView1"
	        android:layout_width="80dp"
	        android:layout_height="80dp"
	        android:scaleType="centerCrop"
	        android:src="@drawable/ic_addpic" />

	</RelativeLayout>

## Demo下载 ##

Github:[https://github.com/ryanlijianchang/TestUpload](https://github.com/ryanlijianchang/TestUpload)


CSDN: [http://download.csdn.net/detail/ljcitworld/9549313](http://download.csdn.net/detail/ljcitworld/9549313)


## 后话 ##

博主只是实现了这一个UI界面，我们开发过程中肯定要实现图片，文字的上传等，这里博主就不再详述了，大家可以看我的另一篇博文[Android学习-使用Async-Http实现图片压缩并上传功能](http://blog.csdn.net/ljcitworld/article/details/51670910)，就这个例子而言，大家如果需要上传多张照片，就可以在添加完照片之后将bitmap存起来，然后通过循环容器的大小，然后每一张图片再上传到服务器即可。还是那句话，**个人能力有限，欢迎大家一起交流学习，我也会虚心接纳大家的指教，不喜勿喷。**


## 参考资料 ##

[Android Developer:Loading Large Bitmaps Efficiently](https://developer.android.com/training/displaying-bitmaps/load-bitmap.html)