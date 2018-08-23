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
