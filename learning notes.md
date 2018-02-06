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

### 保存全尺寸照片
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

### 压缩图片
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

### 计算缩放比例
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


## Bluetooth Low Energy 低功耗蓝牙

### BLE权限
```xml
<uses-permission android:name="android.permission.BLUETOOTH"/>
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
```
如果想声明应用程序仅适用于具有BLE功能的设备，添加：
```xml
<uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/>
```
如果想让应用可适用于不支持BLE功能的设备，设置`required="false"`，并在程序运行时判断设备是否支持BLE，有选择性地禁用BLE相关的功能。
```java
if (!getPackageManager().hasSystemFeature(PackageManager.FEATURE_BLUETOOTH_LE)) {
    Toast.makeText(this, R.string.ble_not_supported, Toast.LENGTH_SHORT).show();
    finish();
}
```

### 获得BluetoothAdapter
```java
private BluetoothAdapter mBluetoothAdapter;

final BluetoothManager bluetoothManager = 
        (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
mBluetoothAdapter = bluetoothManager.getAdapter();
```

### 开启蓝牙
```java
/**
 * 如果蓝牙未开启，显示对话框请求开启蓝牙
 * REQUEST_ENABLE_BT：请求码，自定义常量
 */
if (mBluetoothAdapter == null || !mBluetoothAdapter.isEnabled()) {
    Intent enableBtIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
    startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT);
}
```

### 扫描BLE设备
由于扫描过程耗费电池电量：
* 找到所需设备后立即停止扫描
* 为扫描过程设定时间限制，不要一直扫描
```java
/*
 * 扫描并展示可用的BLE设备
 */
public class DeviceScanActivity extends ListActivity {
    private BluetoothAdapter mBluetoothAdapter;
    private boolean mScanning;
    private Handler mHandler;

    // 扫描时间为10秒
    private static final long SCAN_PERIOD = 10000;

    private void scanLeDevice(final boolean enable) {
        if (enable) {
        
            // 经过SCAN_PERIOD时间后执行run方法
            mHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    mScanning = false;
                    // 停止扫描
                    mBluetoothAdapter.stopLeScan(mLeScanCallback);
                }
            }, SCAN_PERIOD);

            mScanning = true;            
            // 开启扫描
            mBluetoothAdapter.startLeScan(mLeScanCallback);
        } else {
            mScanning = false;
            mBluetoothAdapter.stopLeScan(mLeScanCallback);
        }
    }
}
```
> 如果只扫描指定类型的设备，调用`startLeScan(UUID[], BluetoothAdapter.LeScanCallback)`方法。

实现BluetoothAdapter.LeScanCallback接口，接收扫描结果：
```java
private LeDeviceListAdapter mLeDeviceListAdapter;
private BluetoothAdapter.LeScanCallback mLeScanCallback = new BluetoothAdapter.LeScanCallback() {
    @Override
    public void onLeScan(final BluetoothDevice device, int rssi, byte[] scanRecord) {
        runOnUiThread(new Runnable() {
            @Override
            public void run() {
                mLeDeviceListAdapter.addDevice(device);
                mLeDeviceListAdapter.notifyDataSetChanged();
            }
        });
    }
};
```

### 连接GATT SERVER
```java
/*
 * 参数1：Context
 * 参数2：boolean 是否自动连接
 * 参数3：BluetoothGattCallback
 */
BluetoothGatt mBluetoothGatt = device.connectGatt(this, false, mGattCallback);
```
创建一个Service用来和BLE设备进行交互，创建BluetoothGattCallback实例并重写内部方法来传递返回结果
```java
public class BluetoothLeService extends Service {
    private static final String TAG = BluetoothLeService.class.getSimpleName();
	
    private BluetoothManager mBluetoothManager;
    private BluetoothAdapter mBluetoothAdapter;
    private String mBluetoothDeviceAddress;
    private BluetoothGatt mBluetoothGatt;
    private int mConnectionState = STATE_DISCONNECTED;

    private static final int STATE_DISCONNECTED = 0;
    private static final int STATE_CONNECTING = 1;
    private static final int STATE_CONNECTED = 2;

    public final static String ACTION_GATT_CONNECTED = 
            "com.example.bluetooth.le.ACTION_GATT_CONNECTED";
    public final static String ACTION_GATT_DISCONNECTED = 
            "com.example.bluetooth.le.ACTION_GATT_DISCONNECTED";
    public final static String ACTION_GATT_SERVICES_DISCOVERED = 
            "com.example.bluetooth.le.ACTION_GATT_SERVICES_DISCOVERED";
    public final static String ACTION_DATA_AVAILABLE = 
            "com.example.bluetooth.le.ACTION_DATA_AVAILABLE";
    public final static String EXTRA_DATA = "com.example.bluetooth.le.EXTRA_DATA";

    public final static UUID UUID_HEART_RATE_MEASUREMENT = 
            UUID.fromString(SampleGattAttributes.HEART_RATE_MEASUREMENT);

    private final BluetoothGattCallback mGattCallback = new BluetoothGattCallback() {
    
        @Override
        public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
            String intentAction;
            // 连接成功
            if (newState == BluetoothProfile.STATE_CONNECTED) {
                intentAction = ACTION_GATT_CONNECTED;
                mConnectionState = STATE_CONNECTED;
                
                // 发送广播到UI线程进行相应处理
                broadcastUpdate(intentAction);
                Log.i(TAG, "Connected to GATT server.");
                
                // 搜索Service
                Log.i(TAG, "Attempting to start service discovery:" + 
                        mBluetoothGatt.discoverServices());
            } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {         // 连接断开                   
                intentAction = ACTION_GATT_DISCONNECTED;
                mConnectionState = STATE_DISCONNECTED;
                Log.i(TAG, "Disconnected from GATT server.");
                broadcastUpdate(intentAction);
            }
        }

        @Override
        public void onServicesDiscovered(BluetoothGatt gatt, int status) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                broadcastUpdate(ACTION_GATT_SERVICES_DISCOVERED);
            } else {
                Log.w(TAG, "onServicesDiscovered received: " + status);
            }
        }

        @Override
        public void onCharacteristicRead(BluetoothGatt gatt, 
                BluetoothGattCharacteristic characteristic, int status) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                broadcastUpdate(ACTION_DATA_AVAILABLE, characteristic);
            }
        }
}
```

### 接收BLE Notifications
```java
private BluetoothGatt mBluetoothGatt;
BluetoothGattCharacteristic characteristic;
boolean enabled;
...
mBluetoothGatt.setCharacteristicNotification(characteristic, enabled);
...
BluetoothGattDescriptor descriptor = characteristic.getDescriptor(
        UUID.fromString(SampleGattAttributes.CLIENT_CHARACTERISTIC_CONFIG));
descriptor.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);
mBluetoothGatt.writeDescriptor(descriptor);
```
当characteristic发生变化时将会触发onCharacteristicChanged()回调方法
```java
@Override
// Characteristic notification
public void onCharacteristicChanged(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic) {
    broadcastUpdate(ACTION_DATA_AVAILABLE, characteristic);
}
```

### 关闭
```java
public void close() {
    if (mBluetoothGatt == null) {
        return;
    }
    mBluetoothGatt.close();
    mBluetoothGatt = null;
}
```


## BottomNavigationView 底部导航栏
BottomNavigationView为谷歌官方导航控件，可结合ViewPager实现底部导航功能
```xml
<android.support.design.widget.BottomNavigationView
    android:id="@+id/bottom_navigation_view"
    android:layout_width="match_parent"
    android:layout_height="?attr/actionBarSize"
    android:background="@android:color/white"
    app:itemTextColor="@drawable/selector_bottom_navigation"
    app:itemIconTint="@drawable/selector_bottom_navigation"
    app:elevation="@dimen/bottom_navigation_view_elevation"
    app:menu="@menu/menu_bottom_nav_view" />

<android.support.v4.view.ViewPager
    android:id="@+id/view_pager_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```
在res目录下的menu文件夹下创建 menu_bottom_nav_view.xml，添加所需的菜单项，设置图标和文字
```xml
<menu xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:id="@+id/menu_item_hot_topic"
        android:icon="@drawable/ic_topic"
        android:title="@string/hot_topic"/>
    <item android:id="@+id/menu_item_news"
        android:icon="@drawable/ic_news"
        android:title="@string/news"/>
    <item android:id="@+id/menu_item_tech_news"
        android:icon="@drawable/ic_code"
        android:title="@string/developer_news"/>
    <item android:id="@+id/menu_item_more"
        android:icon="@drawable/ic_more"
        android:title="@string/more"/>
</menu>
```
在res目录下的drawable文件夹下创建 selector_bottom_navigation.xml，设置导航菜单在选中和未选中状态下的效果
```xml
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:color="@color/nav_checked" android:state_checked="true" />
    <item android:color="@color/nav_unchecked" android:state_checked="false" />
</selector>
```
在Activity中对控件进行初始化并添加监听事件等
```Java
BottomNavigationView mBottomNavigationView = 
        (BottomNavigationView) findViewById(R.id.bottom_navigation_view);
ViewPager mViewPager = (ViewPager) findViewById(R.id.view_pager_main);

// 为BottomNavigationView控件添加事件监听
mBottomNavigationView.setOnNavigationItemSelectedListener(
        new BottomNavigationView.OnNavigationItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(@NonNull MenuItem item) {
                int currentItem = 0;
                switch (item.getItemId()) {
                    case R.id.menu_item_hot_topic:
                        currentItem = 0;
                        break;
                    case R.id.menu_item_news:
                        currentItem = 1;
                        break;
                    case R.id.menu_item_tech_news:
                        currentItem = 2;
                        break;
                    case R.id.menu_item_more:
                        currentItem = 3;
                        break;
                }

                // 当前显示页面为导航栏点击位置对应页面
                if (mViewPager.getCurrentItem() == currentItem) {
                    // TODO 刷新页面或者回到顶部
                } else {
                    // ViewPager跟随BottomNavigationView进行页面切换
                    mViewPager.setCurrentItem(currentItem);
                }
                return true;
            }
        });

// 为ViewPager添加事件监听
mViewPager.addOnPageChangeListener(new ViewPager.OnPageChangeListener() {
    @Override
    public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {}

    @Override
    public void onPageSelected(int position) {
        // BottomNavigationView跟随ViewPager滑动切换选中菜单项
        mBottomNavigationView.getMenu().getItem(position).setChecked(true);
    }

    @Override
    public void onPageScrollStateChanged(int state) {}
});
```
BottomNavigationView在item数量大于3时默认带有滑动切换效果（ShiftingMode），且未提供代码层级的关闭选项，需要通过反射实现
```Java
/**
 * 关闭BottomNavigationView的ShiftingMode效果
 */
private void disableNavigationShiftMode() {
    menuView = (BottomNavigationMenuView)
            mBottomNavigationView.getChildAt(0);

    try {
        // 使用反射
        Field shiftingMode = menuView.getClass().getDeclaredField("mShiftingMode");
        shiftingMode.setAccessible(true);
        shiftingMode.setBoolean(menuView, false);
        shiftingMode.setAccessible(false);

        for (int i = 0; i < menuView.getChildCount(); i++) {
            BottomNavigationItemView itemView = (BottomNavigationItemView)
                    menuView.getChildAt(i);
            itemView.setShiftingMode(false);
            itemView.setChecked(itemView.getItemData().isChecked());
        }

    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }
}
```



