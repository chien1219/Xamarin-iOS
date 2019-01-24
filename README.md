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
  
    
## Toast Effect  
  
![Toast](https://i.stack.imgur.com/gX37X.png)  
Toast is the bubble message that flash and disappear soon.  
in Xamarin iOS, there are several plugin that support toast with different effect.  
Now, Im gonna talk about two of them I have appied to my project.  
For both platform, the [Dependency Service](https://docs.microsoft.com/xamarin/xamarin-forms/app-fundamentals/dependency-service/introduction) is needed, and the other part as well as interface are on my [Xamarin Android](https://github.com/chien1219/Android-Memory)  
```  
using MyProject.Interface;
using MyProject.iOS.DependencyService;
using GlobalToast;

[assembly: Xamarin.Forms.Dependency(typeof(ToastHelperService))]
namespace MyProject.iOS.DependencyService
{
    public class ToastHelperService : IToastHelper
    {
        // API of Android is fixed with 2 and 3.5 seconds, here for unitary
        const double LONG_DELAY = 3500;
        const double SHORT_DELAY = 2000;

        // 1/5 of screen height margin from buttom.
        float screenHeight = (float)Plugin.XamJam.Screen.CrossScreen.Current.Size.Height / 5;

        public void LongAlert(string message)
        {
            ToastLayout toastLayout = new ToastLayout();
            toastLayout.MarginBottom = screenHeight;
            Toast.MakeToast(message).SetLayout(toastLayout).SetDuration(LONG_DELAY).Show();
        }
        public void ShortAlert(string message)
        {
            ToastLayout toastLayout = new ToastLayout();
            toastLayout.MarginBottom = screenHeight;
            Toast.MakeToast(message).SetLayout(toastLayout).SetDuration(SHORT_DELAY).Show();
        }
    }
}
```  
  
The above code makes the toast message appear on the 1/5 place on the screen from bottom.  
and the nuget [Toast.IOS](https://www.nuget.org/packages/Toast.iOS/) is needed.  
  
The second way is  to use [BTProgressHUD](https://github.com/nicwise/BTProgressHUD)  

  
```
using BigTed;
.
.
.  
BTProgressHUD.ShowToast("Hello from Toast");  
```  
But this way is a little bit constrained, with non-customized background, position (only three choice in its ENUM), so I did not choose this one as my decision since I want to show in the low position but bottom.  
  
    
## ScrollView scroll to top automatically
  
Same as stated in [issue](https://github.com/xamarin/Xamarin.Forms/issues/3465), In our app, when we scroll to bottom of a scrollview and there's some layout is going to be loading, the page, or the scroll view is going to scroll to the top automatically.  
The following code is what we do on scrolled.  
```
private void OnScrolled(object sender, ScrolledEventArgs e)
        {
            ScrollView scrollView = sender as ScrollView;
            double scrollingSpace = scrollView.ContentSize.Height - scrollView.Height;

            if (scrollingSpace <= (e.ScrollY + 10)) // Touched bottom
            {
                AddNewImage();
            }
        }
```
I couldn't found any causing idea with this situation
Hence, I add a ScrollTo actively to reduce the bounce back. Then everything seems to be good.  
Though this is not properly solved but an electic way.  
```
double scrollY = 0;
private void OnScroll(object sender, ScrolledEventArgs e)
{
    if (scrollY != 0 && e.ScrollY == 0)
    {
        MyScrollView.ScrollToAsync(x: 0, y: scrollY + 10, animated: false);
    }
    else
    {
        scrollY = e.ScrollY;
    }
}
```
