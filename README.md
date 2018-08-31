# Xamarin-iOS

This is written at the time that I encounter some problems during my implementation

## Get Image Path when iCloud backup on From PHAsset

It is simple to get **Image Path** after user picked [PHAsset](https://developer.xamarin.com/api/type/Photos.PHAsset/) from gallery
by **FullSizeImageUrl**   
from Class [Photos.PHContentEditingInput](https://developer.xamarin.com/api/type/Photos.PHContentEditingInput/)
 
```
asset.RequestContentEditingInput(options, (input, nsdictionary) =>
{
    NSUrl path = input.FullSizeImageUrl;
    paths.Add(path.Path);
});
```
  
  However, What if the **Full-Size Image** picked by user is stored in the iCloud?
  The `input` is going to be `null` and hence an null-reference exception will be thrown.  
  At this time a null checking or delay will not help.  
  The Following null checking will just ignore the input and never achieve iCloud file.
  
  ```
  asset.RequestContentEditingInput(options, (input, nsdictionary) =>
{
    if (input != null){
        NSUrl path = input.FullSizeImageUrl;
        paths.Add(path.Path);
    }
});
 ```
  
  From the [options](https://developer.xamarin.com/api/type/Photos.PHContentEditingInputRequestOptions/)
  given in the parameter, we can see [NetworkAccessAllowed](https://developer.xamarin.com/api/property/Photos.PHContentEditingInputRequestOptions.NetworkAccessAllowed/) flag.  
  By this flag, we can wait and retrieve not the optimized image but original image from iCloud.
  
   ```
    PHContentEditingInputRequestOptions options = new PHContentEditingInputRequestOptions
    {
        NetworkAccessAllowed = true,
        ProgressHandler = (double progree, ref bool stop) => 
        {
            // Wait and lock app instance by your own loading system
            UserDialogLoadingSystem.ShowLoading(delayTimeSecond: 0, message: AppResources.UPLOADING_MSG, timeout: 20);
        }                    
    };
```

## Get UIImage From PHAsset
The Flag is Also Applied to the way to get UIImage by [RequestImageForAsset](https://developer.xamarin.com/api/member/Photos.PHImageManager.RequestImageForAsset/)

```
var manager = PHImageManager.DefaultManager;
var request = new PHImageRequestOptions()
{
    Synchronous = true,
    ResizeMode = PHImageRequestOptionsResizeMode.None,
    NetworkAccessAllowed = true,
};

manager.RequestImageForAsset(assets[idx], new CoreGraphics.CGSize(assets[idx].PixelWidth, assets[idx].PixelHeight), PHImageContentMode.AspectFill, request, (result, info) =>
{
    rtn = result;
});
```  
  
  
## Crash when handling permission problem  
  
In my work on uploading image, I found a crash that happen when I try to open camera on my iPhone but the permission is denied.  
The [Plugin.media](https://github.com/jamesmontemagno/MediaPlugin) was applied to my code  
  
```
 await CrossMedia.Current.Initialize();  
  
 file = await CrossMedia.Current.TakePhotoAsync(new Plugin.Media.Abstractions.StoreCameraMediaOptions
 {
     Directory = "MyDir",
     Name = $"MyApp{DateTime.UtcNow}.jpg",
     PhotoSize = Plugin.Media.Abstractions.PhotoSize.MaxWidthHeight,
     MaxWidthHeight = 1080,
     CompressionQuality = 92,
 });
 ```  
 The crash happened at the time calling ```TakePhotoAsync``` when camera permission is denied.  
 To solve this problem, I surveyed and work on it for a long time and found [James](https://github.com/jamesmontemagno) in MS has pretty good work on this topic.  
 By [PermissionPlugin](https://github.com/jamesmontemagno/PermissionsPlugin) with latest version (now 3.0.0.12)
 combined with latest [Plugin.media](https://github.com/jamesmontemagno/MediaPlugin)  
 (There are some issues such as [Version](https://github.com/jamesmontemagno/PermissionsPlugin/issues/105) that comes out when the version of two mutually-supported plugin not matched)  
 The following code is my solution:  
   
   ```
public async Task<PermissionStatus> CheckPermissionAsync()
{
   var permissionStatus = await CrossPermissions.Current.CheckPermissionStatusAsync(Permission.Camera);
   
   if (permissionStatus != PermissionStatus.Granted)
   {
       var askResult = await CrossPermissions.Current.RequestPermissionsAsync(Permission.Camera);
       if (askResult.ContainsKey(Permission.Camera))
       {
           permissionStatus = askResult[Permission.Camera];
       }
   }
   else
   {
       return permissionStatus;
   }

    if (permissionStatus != PermissionStatus.Granted)
    {
        bool result = await App.Instance.MainPage.DisplayAlert("No camera permission", "", "Go setting", "Cancel");
        if (result)
        {
            CrossPermissions.Current.OpenAppSettings();
        }
     }

   return permissionStatus;
}
  ```  
  
and you can use it like:  
  
```
var status = await CheckPermissionAsync();
            
if (status == PermissionStatus.Granted)
{
    // Your capture method
}
```  
  
I found a interesting thing when debugging, iOS system will not ask second time if you deny the access.  
Hence, By checking permission and redirect to setting page could solve the problem on both platform (in Xamarin.Forms)  
There are some additional issues came out when using those plugins, such as [CurrentActivity](https://github.com/jamesmontemagno/CurrentActivityPlugin/blob/master/README.md) and some API usage.  
However, the [PermissionPlugin](https://github.com/jamesmontemagno/PermissionsPlugin) has stated clearly.  
  
among them CurrentActivity is interesting for use code below in OnCreate(), I think it's a great way for not only debug bug some specific content usage.  
```  
Plugin.CurrentActivity.CrossCurrentActivity.Current.Init(this, bundle);
```  
