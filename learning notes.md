## Retrofit结合RxJava

```Java
// 创建接口
public interface GetRequest_Interface {
    @GET("url地址")
    Obserable<Bean> getCall();
}

Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("http://www.baidu.com")
    .addConverterFactory(GsonConverterFactory.create())  // 使用Gson解析
    .addCallAdapterFactory(RxJava2CallAdapterFactory.create())  // 支持RxJava
    .build();

// 创建接口实例
GetRequest_Interface request = retrofit.create(GetRequest_Interface.class);

Observable<Bean> observable = request.getCall();

// 发送网络请求
observable.subscribeOn(Schedulers.io())  // 在IO线程请求网络
    .observeOn(AndroidSchedulers.mainThread())  // 在主线程处理请求结果
    .subscribe(new Observer<Bean> {
			  
        // 发送请求后调用（无论请求是否成功）
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe");
        }
					
        // 发送请求成功后调用
        @Override
        public void onNext(Bean result) {
            // TODO 对result进行处理
        }

        // 请求成功后，onNext()后调用
        @Override 
        public void onComplete() {
            Log.d(TAG, "请求成功");
        }
        
        // 请求失败后调用
        @Override
        public void onError(Throwable e) {
            Log.d(TAG, "请求失败");
        }
    });
```


## AsyncTask

```Java
/**
 * 参数1 Params：执行AsyncTask时要传入的参数，可用于在后台任务中使用
 * 参数2 Progress：进度单位
 * 参数3 Result：返回值类型
 */
class DownloadTask extends AsyncTask<Void, Integer, Boolean> {
    @Override
    protected void onPreExecute() {
        // 界面的初始化操作
        progressDialog.show();
    }
    
    @Override
    protected Boolean doInBackground(Void... params) {
        // 在子线程中执行，处理耗时操作
	try {
	    while (true) {
	        int downloadPercent = doDownload();
		
		// 反馈当前任务的执行进度
		publishProgres(downloadPercent);
		
		if (downloadPercent >= 100) {
		    break;
		}
	    }
	} catch (Exception e) {
	    return false;
	}
	return true;
    }
    
    @Override
    protected void onProgressUpdate(Integer... values) {
        // 更新UI
	progressDialog.setMessage("Download " + values[0] + "%");
    }
    
    @Override
    protected void onPostExecute(Boolean result) {
        // 后台任务return后执行
        progressDialog.dismiss();
	if (result) {
	    Toast.makeText(context, "Download Success", Toast.LENGTH_SHORT).show();
	} else {
	    Toast.makeText(context, "Download Failed", Toast.LENGTH_SHORT).show();
	}
    }
}
```


## 调用系统相机

```Java
static final int REQUEST_IMAGE_CAPTURE = 1;
private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    
    /**
     * resolveActivity()方法会返回能处理该Intent的第一个Activity
     * 用来检查有没有能处理这个Intent的Activity
     */
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        startActivityForResult(takePictureIntent, REQUEST_IMAGE_CAPTURE);
    }
}
```
```Java
/**
 * 获取照片的缩略图
 * 缩略图适用于作为图标
 */
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
        Bundle extras = data.getExtras();
        Bitmap imageBitmap = (Bitmap) extras.get("data");
        mImageView.setImageBitmap(imageBitmap);
    }
}
```
#### 保存全尺寸照片
```Java
String mCurrentPhotoPath;
private File createImageFile() throws IOException {
    // 创建文件名
    String timeStamp = new SimpleDateFormat("yyyyMMdd_HHmmss").format(new Date());
    String imageFileName = "JPEG_" + timeStamp + "_";
    // 获取适用于存储公共图片的目录，可被所有应用共享
    File storageDir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES);
    File image = File.createTempFile(
        imageFileName,      /* prefix */
        ".jpg",             /* suffix */
        storageDir          /* directory */
    );
    
    mCurrentPhotoPath = "file:" + image.getAbsolutePath();
    return image;
}
```
```Java
static final int REQUEST_TAKE_PHOTO = 1;
private void dispatchTakePictureIntent() {
    Intent takePictureIntent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    
    if (takePictureIntent.resolveActivity(getPackageManager()) != null) {
        File photoFile = null;
        try {
            photoFile = createImageFile();
        } catch (IOException e) {}
        
        if (photoFile != null) {
            takePictureClient.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(photoFile));
            startActivityForResult(takePictureIntent, REQUEST_TAKE_PHOTO);
        }
    }
}
```
#### 压缩图片
```Java
private void setPic() {
    int targetW = mImageView.getWidth();
    int targetH = mImageView.getHeight();
    
    BitmapFactory.Options bmOptions = new BitmapFactory.Options();
    bmOptions.inJustDecodeBounds = true;
    BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    int photoW = bmOptions.outWidth;
    int photoH = bmOptions.outHeight;
    
    int scaleFactor = Math.min(photoW / targetW, photoH / targetH);
    
    bmOptions.inJustDecodeBounds = false;
    bmOptions.imSampleSize = scaleFactor;
    bmOptions.inPurgeable = true;
    Bitmap bitmap = BitmapFactory.decodeFile(mCurrentPhotoPath, bmOptions);
    mImageView.setImageBitmap(bitmap);
}
```
#### 计算缩放比例
```Java
public static int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHeight) {
    final int height = options.outHeight;
    final int width = options.outWidth;
    int inSampleSize = 1;
    
    if (height > reqHeight || width > reqWidth) {
        final int halfHeight = height / 2;
        final int halfWidth = height / 2;
        
        while ((halfHeight / inSampleSize) > reqHeight && (halfWidth / inSampleSize) > reqWidth) {
            inSampleSize *= 2;
        }
    }
    
    return inSampleSize;
}
```
## 十六进制字符串与字节数组之间的转换
```Java
/**
 * 数组转字符串
 * {0xF0, 0x08} -> "F008"
 */
public static String bytesToHexString(byte[] bytes) {
    StringBuilder sb = new StringBuilder();
    for (byte value : bytes) {
        int v = value & 0xFF;
        String str = Integer.toHexString(v);
        if (str.length() < 2) {
            sb.append("0");
        }
        sb.append(str);
    }

    return sb.toString();
}
```
```Java
/*
 * 字符串转数组
 */
public static byte[] hexStringToBytes(String hexString) {
    hexString = hexString.toUpperCase();
    char[] charArray = hexString.toCharArray();
    int length;
    byte[] bytes;

    if (hexString.length() % 2 == 0) {
        length = hexString.length() / 2;
        bytes = new byte[length];
        bytes[0] = (byte) (charToByte(charArray[0]) << 4 | charToByte(charArray[1]));

        for (int i = 1; i < length; i++) {
            int pos = i * 2;
            bytes[i] = (byte) (charToByte(charArray[pos]) << 4 | charToByte(charArray[pos + 1]));
        }
    } else {
        length = hexString.length() / 2 + 1;
        bytes = new byte[length];
        bytes[0] = (byte) charToByte(charArray[0]);

        for (int i = 1; i < length; i++) {
            int pos = i * 2 - 1;
            bytes[i] = (byte) (charToByte(charArray[pos]) << 4 | charToByte(charArray[pos + 1]));
        }
    }

    return bytes;
} 

private static byte charToByte(char c) {
    return (byte) "0123456789ABCDEF".indexOf(c);
}
```



