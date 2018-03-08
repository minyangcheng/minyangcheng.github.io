---
title: android调用系统相机和相册选择图片
date: 2016-11-28 21:26:21
tags: android
---
项目中有一个功能让用户选择图片并上传服务器，在网上看了一些人写的工具类库，有点不符合自己的心意，于是自己写了一个调用系统相机或相册选择图片的工具类。

>源码地址 : <https://github.com/minyangcheng/ImageSelector>

<!-- more -->

## 实现思路

1. 调用系统相机和相册的api，获取图片的路径。
2. 将获取的图片复制到自定义限制文件个数的缓存中，防止多次拍照后图片存放本地，增加app占用的硬盘空间。从而避免了在外部需要手动释放图片所占的硬盘空间。

## 实现代码

### 如何使用?

```java
@OnClick(R.id.btn_select_image)
void clickSelectImageIv(){
    String[] items={"拍照","图库"};
    AlertDialog.Builder builder=new AlertDialog.Builder(this)
            .setTitle("选择照片方式")
            .setItems(items, new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    if(which==0){
                        EasyImageSelector.getInstance(getActivity()).openCamera(getActivity());
                    }else{
                        EasyImageSelector.getInstance(getActivity()).openGalleryPicker(getActivity());
                    }
                }
            });
    builder.show();
}

private AppCompatActivity getActivity(){
    return this;
}

@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);
    EasyImageSelector.getInstance(getActivity()).handleActivityResult(requestCode, resultCode, data, this, new EasyImageSelector.Callbacks() {
        @Override
        public void onImagePickerError(Exception e, EasyImageSelector.ImageSource source) {
            L.d(TAG, "onImagePickerError exception=%s", e);
        }

        @Override
        public void onImagePicked(final File imageFile, EasyImageSelector.ImageSource source) {
            L.d(TAG, "onImagePicked file=%s", imageFile.getAbsolutePath());
            final File outFile=EasyImageSelector.getInstance(getActivity()).getImageFile();
            if(ImageUtils.compressImageToFile(imageFile, outFile, 500, 500)){
                ImageLoaderWrap.displayFileImage(imageFile, mImageIv);
                new Thread(){
                    @Override
                    public void run() {
                        mOriginalImageFile=imageFile;
                        mCompressedImageFile=outFile;
                        setExifInfoInImageFile(mCompressedImageFile);

                        uploadImageFile(outFile);
                    }
                }.start();
            }else{
                onImagePickerError(new Exception("图片压缩失败"),source);
            }
        }
    });
}
```

### EasyImageSelector对外开放两个方法

```java
public void openCamera(Object obj) {
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    File image = getImageFile();
    Uri capturedImageUri = Uri.fromFile(image);
    intent.putExtra(MediaStore.EXTRA_OUTPUT, capturedImageUri);
    mCameraImageUriStr = capturedImageUri.toString();
    if (obj instanceof Activity) {
        Activity activity = (Activity) obj;
        activity.startActivityForResult(intent, REQ_PICK_PICTURE_FROM_CAMERA);
    } else if (obj instanceof Fragment){
        Fragment fragment=(Fragment) obj;
        fragment.startActivityForResult(intent, REQ_PICK_PICTURE_FROM_CAMERA);
    }
}

public void openGalleryPicker(Object obj) {
    Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
    intent.setType("image/*");
    if (obj instanceof Activity) {
        Activity activity = (Activity) obj;
        activity.startActivityForResult(intent, REQ_PICK_PICTURE_FROM_GALLERY);
    } else if (obj instanceof Fragment){
        Fragment fragment=(Fragment) obj;
        fragment.startActivityForResult(intent, REQ_PICK_PICTURE_FROM_GALLERY);
    }
}
```

### 显示图片数量的缓存类

主要是利用文件的最后修改时间进行排序，如果文件数量到达上限，就删除掉修改时间最早的文件。

```java
public class AmountLimitCache {

    private File mCacheDir;
    private int mMaxAmount=100;  //默认支持100个文件

    public AmountLimitCache(String cacheDir){
        if(TextUtils.isEmpty(cacheDir)){
            throw new IllegalArgumentException("cacheDir can not be null");
        }
        mCacheDir=new File(cacheDir);
        if(!mCacheDir.exists()){
            mCacheDir.mkdir();
        }
    }

    public boolean save(InputStream is , String fileName){
        synchronized (this){
            deleteFilesWhenArriveMaxCount();

            File file=getFile(fileName);
            return FileUtils.writeFile(file, is);
        }
    }

    public boolean save(File inFile , String fileName){
        synchronized (this){
            deleteFilesWhenArriveMaxCount();
            File file=getFile(fileName);
            return FileUtils.copyFile(inFile.getAbsolutePath(), file.getAbsolutePath());
        }
    }

    public void delete(String fileName){
        synchronized (this){
            File file=getFile(fileName);
            if(file.exists()&&file.getParentFile().getAbsolutePath().equals(mCacheDir.getAbsolutePath())){
                file.delete();
            }
        }
    }

    public void clear(){
        synchronized (this){
            File[] files=mCacheDir.listFiles();
            if(files!=null&&files.length>0){
                for(File file : files){
                    file.delete();
                }
            }
        }
    }

    public void setCacheSize(int maxAmount){
        this.mMaxAmount=maxAmount;
    }

    public File getFile(String fileName){
        return new File(mCacheDir,fileName);
    }

    public File[] getFiles(){
        return mCacheDir.listFiles();
    }

    public void deleteFilesWhenArriveMaxCount(){
        File[] files=mCacheDir.listFiles();
        if(files!=null&&files.length>=mMaxAmount){
            Arrays.sort(files, new FileComprator());
            int tempCnt=files.length-(mMaxAmount-1);
            for (int i=0;i<tempCnt;i++){
                files[i].delete();
            }
        }
    }

    public static class FileComprator implements Comparator<File> {
        @Override
        public int compare(File lhs, File rhs) {
            return (int) (lhs.lastModified()-rhs.lastModified());
        }
    }

}
```

## 总结

如果你的app对图片选择没有特别的要求，推荐直接使用调用系统相册或相机，不过这样做的话就不能很好的进行多图选择了，要想拥有多图选择功能还是需要你自定义图片选择器。我在下一篇文章将会介绍我写的自定义图片选择器。