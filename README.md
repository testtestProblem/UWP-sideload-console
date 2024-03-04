# Why need UWP extention
Universal Windows Platform (UWP). It has delicate UI by XAML. But it can not support a lot of WIN32 function. That is why we need UWP extention to unlimited it ability. 

# Project describe
* Paltform: visual studio 2019  
* Function describe: UWP through Windows Desktop Extensions for the UWP tool side loading console.  
* Compile sample: UWP can not run at any CPU, and target should set Windows Application Packaging Project.  
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/b60b14bc-4c7e-4bba-9e72-c146665b3147)

# Just sideload other app

* This UWP_extention can side loading winform by using ```FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync()```.  
 
* Modify WapProj/Package.appxmanifest by adding extensions in Windows Application Packaging Project.
```XML
<Package
  xmlns="http://schemas.microsoft.com/appx/manifest/foundation/windows10"
  xmlns:uap="http://schemas.microsoft.com/appx/manifest/uap/windows10"
  xmlns:rescap="http://schemas.microsoft.com/appx/manifest/foundation/windows10/restrictedcapabilities"
  
  xmlns:mp="http://schemas.microsoft.com/appx/2014/phone/manifest"
  xmlns:desktop="http://schemas.microsoft.com/appx/manifest/desktop/windows10"
  IgnorableNamespaces="uap mp desktop rescap">
```

```XML
 <Applications>
    <Application Id="App"
      Executable="$targetnametoken$.exe"
      EntryPoint="$targetentrypoint$">
      <uap:VisualElements
        DisplayName="WapProj_HotTab_Win10"
        Description="WapProj_HotTab_Win10"
        BackgroundColor="transparent"
        Square150x150Logo="Images\Square150x150Logo.png"
        Square44x44Logo="Images\Square44x44Logo.png">
        <uap:DefaultTile Wide310x150Logo="Images\Wide310x150Logo.png" />
        <uap:SplashScreen Image="Images\SplashScreen.png" />
      </uap:VisualElements>
		
	 <Extensions>
		 <uap:Extension Category="windows.appService">
		 	<uap:AppService Name="SampleInteropService" />
		 </uap:Extension>
		 <desktop:Extension Category="windows.fullTrustProcess" Executable="CollectDataAP\CollectDataAP.exe" />
	 </Extensions>
		
    </Application>
  </Applications>
```

In WapProj/Dependencies 
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/8e778761-9ca0-47d2-8d8f-a4fe259f3b55)

Add UWP extension in UWP/Reference 
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/c3a48b03-6178-4231-be48-334583b6ae3d)
 
Add and cover this in UWP/App.xmal.cs
```C#
sealed partial class App : Application
    {
        public static BackgroundTaskDeferral AppServiceDeferral = null;
        public static AppServiceConnection Connection = null;
        public static bool IsForeground = false;
        public static event EventHandler<AppServiceTriggerDetails> AppServiceConnected;
        public static event EventHandler AppServiceDisconnected;



        private void App_LeavingBackground(object sender, LeavingBackgroundEventArgs e)
        {
            IsForeground = true;
        }

        private void App_EnteredBackground(object sender, EnteredBackgroundEventArgs e)
        {
            IsForeground = false;
        }

        /// <summary>
        /// Initializes the singleton application object.  This is the first line of authored code
        /// executed, and as such is the logical equivalent of main() or WinMain().
        /// </summary>
        public App()
        {
            this.InitializeComponent();
            this.Suspending += OnSuspending;
            this.EnteredBackground += App_EnteredBackground;
            this.LeavingBackground += App_LeavingBackground;
        }

        /// <summary>
        /// Handles connection requests to the app service
        /// </summary>
        protected override void OnBackgroundActivated(BackgroundActivatedEventArgs args)
        {
            base.OnBackgroundActivated(args);

            if (args.TaskInstance.TriggerDetails is AppServiceTriggerDetails details)
            {
                // only accept connections from callers in the same package
                if (details.CallerPackageFamilyName == Package.Current.Id.FamilyName)
                {
                    // connection established from the fulltrust process
                    AppServiceDeferral = args.TaskInstance.GetDeferral();
                    args.TaskInstance.Canceled += OnTaskCanceled;

                    Connection = details.AppServiceConnection;
                    AppServiceConnected?.Invoke(this, args.TaskInstance.TriggerDetails as AppServiceTriggerDetails);
                }
            }
        }

        /// <summary>
        /// Task canceled here means the app service client is gone
        /// </summary>
        private void OnTaskCanceled(IBackgroundTaskInstance sender, BackgroundTaskCancellationReason reason)
        {
            AppServiceDeferral?.Complete();
            AppServiceDeferral = null;
            Connection = null;
            AppServiceDisconnected?.Invoke(this, null);
        }

....

```

* launch app and re-launch when disconnection
Add and cover in UWP/MainPage.xmal.cs
```C#
 public MainPage()
        {
            this.InitializeComponent();

           // ApplicationView.PreferredLaunchViewSize = new Size(200, 200);
           // ApplicationView.PreferredLaunchWindowingMode = ApplicationViewWindowingMode.PreferredLaunchViewSize;
        }

        protected async override void OnNavigatedTo(NavigationEventArgs e)
        {
            base.OnNavigatedTo(e);

            if (ApiInformation.IsApiContractPresent("Windows.ApplicationModel.FullTrustAppContract", 1, 0))
            {
                //for connect or disconnect event
                //App.AppServiceConnected += MainPage_AppServiceConnected;
                //App.AppServiceDisconnected += MainPage_AppServiceDisconnected;

                //for sideload app
                await FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync();
            }
        }

        /// <summary>
        /// When the desktop process is disconnected, reconnect if needed
        /// </summary>
        private async void MainPage_AppServiceDisconnected(object sender, EventArgs e)
        {
            await Dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =>
            { 
                Reconnect();
            });
        }

        /// <summary>
        /// Ask user if they want to reconnect to the desktop process
        /// </summary>
        private async void Reconnect()
        {
            if (App.IsForeground)
            {
                MessageDialog dlg = new MessageDialog("Connection to desktop process lost. Reconnect?");
                UICommand yesCommand = new UICommand("Yes", async (r) =>
                {
                    await FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync();
                });
                dlg.Commands.Add(yesCommand);
                UICommand noCommand = new UICommand("No", (r) => { });
                dlg.Commands.Add(noCommand);
                await dlg.ShowAsync();
            }
        }

.......

```

* if can not find entrypoint, can see this for more information  
https://stackoverflow.com/questions/77065216/dep0700-registration-of-the-app-failed-while-uwp-extension-launch-consult-app/77493220#77493220


# Sideload app and UWP can change message in two way

Go to BelaunchedApp/reference and add those file 
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/9f198e02-c419-4764-a2b8-77e8e318e154)

* It can send data and receive data  
Add and cover sildload_app/program  
```C#
class Program
    { 
        static private AppServiceConnection connection = null; 

        static void Main(string[] args)
        {
            InitializeAppServiceConnection();

            string s_res;

            Console.WriteLine("[1] send data to UWP");
            while ((s_res = Console.ReadLine()) != "")
            {
                int i_res=int.Parse(s_res);

                if (i_res == 1)
                {
                    Send2UWP_2("Hi!", "UWP");
                }
            }
        }

        /// <summary>
        /// Open connection to UWP app service
        /// </summary>
        static private async void InitializeAppServiceConnection()
        {
            connection = new AppServiceConnection();
            connection.AppServiceName = "SampleInteropService";
            connection.PackageFamilyName = Package.Current.Id.FamilyName;
            connection.RequestReceived += Connection_RequestReceived;
            connection.ServiceClosed += Connection_ServiceClosed;

            AppServiceConnectionStatus status = await connection.OpenAsync();
            if (status != AppServiceConnectionStatus.Success)
            {
                //In console, something wrong
                Console.WriteLine(status.ToString());
                //In app, something went wrong ...
                //MessageBox.Show(status.ToString());
                //this.IsEnabled = false;
            }
        }

        /// <summary>
        /// Handles the event when the desktop process receives a request from the UWP app
        /// </summary>
        static private async void Connection_RequestReceived(AppServiceConnection sender, AppServiceRequestReceivedEventArgs args)
        {
            // retrive the reg key name from the ValueSet in the request
            /*
            string key = args.Request.Message["KEY"] as string;
            if (key == "abcd")
            {
                // compose the response as ValueSet
                ValueSet response = new ValueSet();
                response.Add("GOOD", "KEY IS FOUNDED ^^");

                // send the response back to the UWP
                await args.Request.SendResponseAsync(response);
            }
            else
            {
                ValueSet response = new ValueSet();
                response.Add("ERROR", "INVALID REQUEST");
                await args.Request.SendResponseAsync(response);
            }
            */
            int? key1 = args.Request.Message["KEY1"] as int?;
            int? key2 = args.Request.Message["KEY2"] as int?;

            if (key1 != null && key2 != null)
            {
                int ans = (int)key1 + (int)key2;

                // compose the response as ValueSet
                ValueSet response = new ValueSet();
                response.Add("KEY3", ans);

                // send the response back to the UWP
                await args.Request.SendResponseAsync(response);
            }
        }

        /// <summary>
        /// Handles the event when the app service connection is closed
        /// </summary>
        static private void Connection_ServiceClosed(AppServiceConnection sender, AppServiceClosedEventArgs args)
        {
            //In console, connection to the UWP lost, so we shut down the desktop process
            Console.WriteLine("UWP lost connection! Please restart.");
            Console.ReadLine();
            Environment.Exit(0);
            //In app, connection to the UWP lost, so we shut down the desktop process
            //Dispatcher.BeginInvoke(DispatcherPriority.Normal, new Action(() =>
            //{
            //    Application.Current.Shutdown();
            //}));
        }

        static private async void Send2UWP(double a, double b)
        {
            // ask the UWP to calculate d1 + d2
            ValueSet request = new ValueSet();
            request.Add("D1", a);
            request.Add("D2", b);

            //start sending
            AppServiceResponse response = await connection.SendMessageAsync(request);
            //get response
            double result = (double)response.Message["RESULT"];

            Console.WriteLine(result.ToString());
        }

        static private async void Send2UWP_2(string a, string b)
        {
            // ask the UWP to calculate d1 + d2
            ValueSet request = new ValueSet();
            request.Add("s_a", a);
            request.Add("s_b", b);

            //start sending
            AppServiceResponse response = await connection.SendMessageAsync(request);
            //get response
            string result = response.Message["toConsole_result"] as string;

            Console.WriteLine("send data_a to UWP: " + a);
            Console.WriteLine("send data_b to UWP: " + b);
            Console.WriteLine("getting data from UWP: " + result);
        }
    }
```

> [!TIP]
> Also can try this if slected multiple keys

in sildload_app/program
```C#
        /// <summary>
        /// Handles the event when the desktop process receives a request from the UWP app
        /// </summary>
        private async void Connection_RequestReceived(AppServiceConnection sender, AppServiceRequestReceivedEventArgs args)
        { 
            Console.WriteLine("Connection_RequestReceived");

            foreach(object key in args.Request.Message.Keys)
            {
                if((string)key == "BasicInfo")
                {
                    Console.WriteLine("BasicInfo");

                    GetInformation getInformation = new GetInformation();
                    getInformation.InitializeWMIHandler();

                    if (getInformation.InitGlobalVariable() == 0)   //have errpr
                    {
                        // compose the response as ValueSet
                        ValueSet response = new ValueSet();

                        response.Add("BasicInfo2UWP", "Error! Please restart.");

                        // send the response back to the UWP
                        await args.Request.SendResponseAsync(response);
                    }
                    else
                    {
                        string basicInfo = getInformation.GetInfomation();
                        // compose the response as ValueSet
                        ValueSet response = new ValueSet();

                        response.Add("BasicInfo2UWP", basicInfo);

                        // send the response back to the UWP
                        await args.Request.SendResponseAsync(response);
                    }
                }
                else if((string)key == "deviceConfig")
                {
                    DeviceState deviceState = new DeviceState();
                    uint? deviceCode;
                    
                    uint state = deviceState.GetDeviceStatePower();
                    deviceCode = args.Request.Message["deviceConfig"] as uint?;
                    try
                    {
                        foreach (uint device in Enum.GetValues(typeof(DeviceState.DeviceStatePower)))
                        {
                            if ((deviceCode & device) == device)
                            {
                                state = state ^ (uint)deviceCode;
                                deviceState.SetDeviceStatePower(state);
                            }
                        }
                    }
                    catch (Exception e)
                    {
                        //TODO: not verify
                        var dialog = new MessageDialog(e.Message);
                        await dialog.ShowAsync();
                    }
                    // compose the response as ValueSet
                    ValueSet response = new ValueSet();

                    state = deviceState.GetDeviceStatePower();
                    response.Add("res_deviceConfig", state);

                    // send the response back to the UWP
                    await args.Request.SendResponseAsync(response);
                }
	}
}
```
-------------------------------------------
Add and cover UWP/MainPage.xmal  
```XML
<Grid>
        <Button  x:Name="calc_btn"   Margin="49,100,0,0" VerticalAlignment="Top" Click="calc_btn_Click" Height="58" Width="117">
            <TextBlock>calculate<LineBreak/>in sideload app</TextBlock>
        </Button>
        <TextBlock x:Name="puse_textBox" HorizontalAlignment="Left" Margin="124,42,0,0" Text="+" TextWrapping="Wrap" VerticalAlignment="Top" Width="33"/>
        <TextBox x:Name="a_textBlock" HorizontalAlignment="Left" Margin="44,38,0,0" Text="10" TextWrapping="Wrap" VerticalAlignment="Top"/>
        <TextBox x:Name="b_textBlock" HorizontalAlignment="Left" Margin="164,38,0,0" Text="20" TextWrapping="Wrap" VerticalAlignment="Top"/>
        <TextBlock x:Name="equal_textBox" HorizontalAlignment="Left" Margin="244,42,0,0" Text="=" TextWrapping="Wrap" VerticalAlignment="Top" Width="33"/>
        <TextBox x:Name="ans_textBox" HorizontalAlignment="Left" Margin="285,15,0,0" Text="" TextWrapping="Wrap" VerticalAlignment="Top" Width="238" Height="100"/>
        <TextBox x:Name="textBox" HorizontalAlignment="Left" Margin="54,197,0,0" Text="TextBox" TextWrapping="Wrap" VerticalAlignment="Top"/>

</Grid>
```

* It can send data and receive data  
Add and cover UWP/MainPage.xmal.cs
```C#
/// <summary>
    /// An empty page that can be used on its own or navigated to within a Frame.
    /// </summary>
    public sealed partial class MainPage : Page
    {
        public MainPage()
        {
            this.InitializeComponent();

            // ApplicationView.PreferredLaunchViewSize = new Size(200, 200);
            // ApplicationView.PreferredLaunchWindowingMode = ApplicationViewWindowingMode.PreferredLaunchViewSize;
        }

        protected async override void OnNavigatedTo(NavigationEventArgs e)
        {
            base.OnNavigatedTo(e);

            if (ApiInformation.IsApiContractPresent("Windows.ApplicationModel.FullTrustAppContract", 1, 0))
            {
                //for connect or disconnect event
                App.AppServiceConnected += MainPage_AppServiceConnected;
                App.AppServiceDisconnected += MainPage_AppServiceDisconnected;

                //ApplicationData.Current.LocalSettings.Values["parameters"] = "test";

                //for sideload app
                await FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync();
            }
        }


        /// <summary>
        /// When the desktop process is connected, get ready to send/receive requests
        /// </summary>
        private async void MainPage_AppServiceConnected(object sender, AppServiceTriggerDetails e)
        {
            App.Connection.RequestReceived += AppServiceConnection_RequestReceived;
            await Dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =>
            {
                // enable UI to access  the connection
                //btnRegKey.IsEnabled = true;
            });
        }

        /// <summary>
        /// Handle calculation request from desktop process
        /// (dummy scenario to show that connection is bi-directional)
        /// </summary>
        private async void AppServiceConnection_RequestReceived(AppServiceConnection sender, AppServiceRequestReceivedEventArgs args)
        {
            //get value from sideload app
            string s_a = (string)args.Request.Message["s_a"];
            string s_b = (string)args.Request.Message["s_b"];
            //double result = d1 + d2;

            //send "have get data" to sideload app
            ValueSet response = new ValueSet();
            response.Add("toConsole_result", "Hello! sideload app");
            await args.Request.SendResponseAsync(response);

            //textBox.Text = ""; //it will error
            //log the getting value in the UI for demo purposes
            await this.Dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =>     //I don't know this code
            {
                textBox.Text += string.Format("Request(getting data from sideload app):\ndata 1: {0} \ndata 2: {1}", s_a, s_b);
            });
        }


        /// <summary>
        /// When the desktop process is disconnected, reconnect if needed
        /// </summary>
        private async void MainPage_AppServiceDisconnected(object sender, EventArgs e)
        {
            await Dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =>
            {
                Reconnect();
            });
        }

        /// <summary>
        /// Ask user if they want to reconnect to the desktop process
        /// </summary>
        private async void Reconnect()
        {
            if (App.IsForeground)
            {
                MessageDialog dlg = new MessageDialog("Connection to desktop process lost. Reconnect?");
                UICommand yesCommand = new UICommand("Yes", async (r) =>
                {
                    await FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync();
                });
                dlg.Commands.Add(yesCommand);
                UICommand noCommand = new UICommand("No", (r) => { });
                dlg.Commands.Add(noCommand);
                await dlg.ShowAsync();
            }
        }

        private async void calc_btn_Click(object sender, RoutedEventArgs e)
        {
            //send value to sideloade app
            ValueSet request = new ValueSet();
            request.Add("KEY1", int.Parse(a_textBlock.Text));
            request.Add("KEY2", int.Parse(b_textBlock.Text));
            AppServiceResponse response = await App.Connection.SendMessageAsync(request);   //send data and get response 

            //display the response key/value pairs
            //tbResult.Text = "";
            foreach (string key in response.Message.Keys)
            {
                ans_textBox.Text = "sended by sideload app\nkey: " + key + "\nvalue: " + response.Message[key];
            }
        }
    }
```


# Package to app

* First should generate self-signed certification for UWP and WapProj.  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/f4a73f94-975d-495e-b24e-ab5096e5db7c)    
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/31e7fbd3-d36b-475c-a25e-46301a8d2791)  

* Add picture
If use default picture may will error
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/42b275c0-13a5-4fe6-8154-fb7d43cf36c3)  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/980e7e93-62d9-44f6-8a10-09845e616006)  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/dc126578-9dab-4003-ba56-d4d768c81b68)  

* Package
In WapProj  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/28a0eca9-dff0-414c-948b-f2e22d5bbfee)  
next  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/a6ade578-dcce-4cce-ace2-269bf43ee061)  
next  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/70903eaa-5757-4aed-a45d-f482c740bcf5)  
x64 Release
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/0ccf524a-e871-41b0-9044-51e5b63be3d2)  

* Sign  
Go to C:\...\HotTab_Win10\WapProj_HotTab_Win10\bin\x64\Release  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/4848c730-a3da-40e8-be84-002866f92b9e)  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/211e0a07-2397-41aa-b288-e5eb296f59eb)  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/77b53f39-26aa-4993-ae43-016e578e30ff)  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/cf23ebf1-6539-4894-9874-b02dc585f770)  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/30bc3f51-900d-476d-a791-764be9724494)  
Add below 3 picture sign  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/e36b8e5d-b5f3-4ee5-8afb-235c7c6bfc2d)  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/eea63ec4-3735-47f0-81b1-77fa9d6f5ec9)  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/9df4efa5-bb4f-49ae-aa2c-716ac369745b)  

Or, It will show error code 0x800B010A  
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/fa541821-e83a-4886-8095-91189c6f11b8)  

# Other

error code => DEP1000: Cannot copy file  
Just close all debug window
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/1654822f-431d-4822-87f5-7c9279c98887)


# Reference
* https://stefanwick.com/2018/04/16/uwp-with-desktop-extension-part-3/
* https://www.cnblogs.com/manupstairs/p/14178931.html
* https://github.com/StefanWickDev/UWP-FullTrust/tree/master/UWP_FullTrust_3
* https://freetion26.medium.com/uwp%E9%80%8F%E9%81%8Eappserviceconnection%E5%92%8Cwin32-process%E6%BA%9D%E9%80%9A%E5%8F%96%E5%BE%97%E7%B3%BB%E7%B5%B1%E8%B7%AF%E5%BE%91%E4%B8%8B%E6%AA%94%E6%A1%88%E5%88%97%E8%A1%A8-981dd0765e52
