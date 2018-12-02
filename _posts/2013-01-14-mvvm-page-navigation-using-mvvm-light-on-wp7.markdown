---
layout: post
title: MVVM Page Navigation Using MVVM Light On WP7
date: '2013-01-14 08:05:05'
---

<p>Page navigation can be a pain to achieve in a true MVVM way, I recently ran in to this problem with a WP7 project but the solution should be the same for Silverlight on Windows and WPF. I ended up solving this using base pages and the messaging functionality in MVVM light. The solution isn't as clean as I would have liked but I’m fairly happy with it.</p> <p>Below are the steps I took to achieve this.</p> <ol> <li>Create a new WP7 Application in Visual Studio and call it MVVMNavigation.  <li>Add the “MMVM Light Libraries Only” Nuget package to the project.  <li>Create a new class called BaseApplicationPage.cs with the following code in it <pre class="brush: csharp; gutter: false; toolbar: false;">using System;
using GalaSoft.MvvmLight.Messaging;
using Microsoft.Phone.Controls;

namespace MVVMNavigation
{
    public class BaseApplicationPage : PhoneApplicationPage
    {
        public void NavigateTo(Uri uri)
        {
            if (uri.ToString() == "/GoBack.xaml")
                NavigationService.GoBack();
            else
                NavigationService.Navigate(uri);
        }

        protected override void OnNavigatedFrom(System.Windows.Navigation.NavigationEventArgs e)
        {
            Messenger.Default.Unregister&lt;Uri&gt;(this);
            base.OnNavigatedFrom(e);
        }

        protected override void OnNavigatedTo(System.Windows.Navigation.NavigationEventArgs e)
        {
            Messenger.Default.Register&lt;Uri&gt;(this, "NavigationRequest", NavigateTo);
            base.OnNavigatedTo(e);
        }
    }
}
</pre>This will register any pages that inherit from the base page to receive navigation messages from view models and will handle them accordingly. It also unregisters each page as it's navigated away from so you don’t end up with lots of message listeners as pages are navigated to and from. 
<li>We then need to change our MainPage to use this BasePage. To do that change the opening and closing tags on MainPage.xaml from phone:PhoneApplicationPage to mvvmnavigation:baseapplicationpage 
<li>Change MainPage.xaml.cs to inherit from BaseApplication page instead of PhoneApplicationPage. <pre class="brush: csharp; gutter: false; toolbar: false;">public partial class MainPage : MVVMNavigation.BaseApplicationPage</pre>
<li>Add a new page called SecondPage and make the same changes you made to the MainPage to SecondPage. 
<li>We’re now in a position where we can create a view model for each page that sends the Navigation message. Create 2 new classes called MainPageViewModel and SecondPageViewModel. <pre class="brush: csharp; gutter: false; toolbar: false;">using System;
using GalaSoft.MvvmLight.Command;
using GalaSoft.MvvmLight.Messaging;

namespace MVVMNavigation
{
    public class MainPageViewModel
    {
        public RelayCommand GotoSecondPage { get; set; }

        public MainPageViewModel()
        {
            GotoSecondPage = new RelayCommand(gotoSecondPage);
        }

        private void gotoSecondPage()
        {
            Messenger.Default.Send(new Uri("/SecondPage.xaml", UriKind.Relative), "NavigationRequest");
        }
    }
}
</pre><pre class="brush: csharp; gutter: false; toolbar: false;">using System;
using GalaSoft.MvvmLight.Command;
using GalaSoft.MvvmLight.Messaging;

namespace MVVMNavigation
{
    public class SecondPageViewModel
    {
        public RelayCommand GoBack { get; set; }

        public SecondPageViewModel()
        {
            GoBack = new RelayCommand(goBack);
        }

        private void goBack()
        {
            Messenger.Default.Send(new Uri("/GoBack.xaml", UriKind.Relative), "NavigationRequest");
        }    
    }
}

</pre>
<li>Now we need a way to access our view models from our pages, for this we are going to use a view model locator. Create a new class called ViewModelLocator.cs and add the following to it. <pre class="brush: csharp; gutter: false; toolbar: false;">namespace MVVMNavigation
{
    public class ViewModelLocator
    {
        public MainPageViewModel MainPageViewModel { get; set; }
        public SecondPageViewModel SecondPageViewModel { get; set; }

        public ViewModelLocator()
        {
            MainPageViewModel = new MainPageViewModel();
            SecondPageViewModel = new SecondPageViewModel();
        }
    }
}
</pre>
<li>We now need to create an instance of ViewModelLocator, to do this open App.xaml and add a namespace/resource to the ViewModelLocator like this.... <pre>&lt;Application 
    x:Class="MVVMNavigation.App"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"       
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:phone="clr-namespace:Microsoft.Phone.Controls;assembly=Microsoft.Phone"
    xmlns:shell="clr-namespace:Microsoft.Phone.Shell;assembly=Microsoft.Phone"
    xmlns:mvvmNavigation="clr-namespace:MVVMNavigation"&gt;

    &lt;!--Application Resources--&gt;
    &lt;Application.Resources&gt;
        &lt;mvvmNavigation:ViewModelLocator x:Key="Locator" /&gt;
    &lt;/Application.Resources&gt;

    &lt;Application.ApplicationLifetimeObjects&gt;
        &lt;!--Required object that handles lifetime events for the application--&gt;
        &lt;shell:PhoneApplicationService 
            Launching="Application_Launching" Closing="Application_Closing" 
            Activated="Application_Activated" Deactivated="Application_Deactivated"/&gt;
    &lt;/Application.ApplicationLifetimeObjects&gt;
&lt;/Application&gt;
</pre>
<li>If you've followed this far you shoud now be able to bind your pages to the view models. Open MainPage.xaml and set the DataContet to MainPageViewModel then add a button with it's command bound to GotoSecondPage. Like this... <pre>&lt;mvvmNavigation:BaseApplicationPage 
    x:Class="MVVMNavigation.MainPage"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:phone="clr-namespace:Microsoft.Phone.Controls;assembly=Microsoft.Phone"
    xmlns:shell="clr-namespace:Microsoft.Phone.Shell;assembly=Microsoft.Phone"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" xmlns:mvvmNavigation="clr-namespace:MVVMNavigation"
    mc:Ignorable="d" d:DesignWidth="480" d:DesignHeight="768"
    FontFamily="{StaticResource PhoneFontFamilyNormal}"
    FontSize="{StaticResource PhoneFontSizeNormal}"
    Foreground="{StaticResource PhoneForegroundBrush}"
    SupportedOrientations="Portrait" Orientation="Portrait"
    shell:SystemTray.IsVisible="True"
    DataContext="{Binding MainPageViewModel, Source={StaticResource Locator}}"&gt;

    &lt;Grid x:Name="LayoutRoot" Background="Transparent"&gt;
        &lt;Grid.RowDefinitions&gt;
            &lt;RowDefinition Height="Auto"/&gt;
            &lt;RowDefinition Height="*"/&gt;
        &lt;/Grid.RowDefinitions&gt;
        &lt;StackPanel x:Name="TitlePanel" Grid.Row="0" Margin="12,17,0,28"&gt;
            &lt;TextBlock Text="Page 1"/&gt;
            &lt;Button Command="{Binding GotoSecondPage}"&gt;Click Me&lt;/Button&gt;
        &lt;/StackPanel&gt;
    &lt;/Grid&gt;
&lt;/mvvmNavigation:BaseApplicationPage&gt;
</pre>
<li>Then do the same to SecondPage by setting the DataContext to SeondPageViewModel and binding the button command to GoBack.... <pre>&lt;mvvmNavigation:BaseApplicationPage
    x:Class="MVVMNavigation.SecondPage"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:phone="clr-namespace:Microsoft.Phone.Controls;assembly=Microsoft.Phone"
    xmlns:shell="clr-namespace:Microsoft.Phone.Shell;assembly=Microsoft.Phone"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" xmlns:mvvmNavigation="clr-namespace:MVVMNavigation"
    FontFamily="{StaticResource PhoneFontFamilyNormal}"
    FontSize="{StaticResource PhoneFontSizeNormal}"
    Foreground="{StaticResource PhoneForegroundBrush}"
    SupportedOrientations="Portrait" Orientation="Portrait"
    mc:Ignorable="d"
    shell:SystemTray.IsVisible="True"
    DataContext="{Binding SecondPageViewModel, Source={StaticResource Locator}}"&gt;

    &lt;Grid x:Name="LayoutRoot" Background="Transparent"&gt;
        &lt;Grid.RowDefinitions&gt;
            &lt;RowDefinition Height="Auto"/&gt;
            &lt;RowDefinition Height="*"/&gt;
        &lt;/Grid.RowDefinitions&gt;
        &lt;StackPanel Grid.Row="0" Margin="12,17,0,28"&gt;
            &lt;TextBlock Text="Page Two"/&gt;
            &lt;Button Command="{Binding GoBack}"&gt;Go Back&lt;/Button&gt;
        &lt;/StackPanel&gt;
    &lt;/Grid&gt;
&lt;/mvvmNavigation:BaseApplicationPage&gt;
</pre></li></ol>
<p>If you now run the project you will be able to run your project and navigate between pages using commands bound to methods on the view model. I think this solution is a lot neater than reverting to code behind in each page to handle navigation. Let me know any thoughts/suggestions you have in the comments…</p>