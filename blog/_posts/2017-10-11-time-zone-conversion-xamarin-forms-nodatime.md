---
title: 'Time zone conversion in Xamarin Forms XAML using NodaTime'
layout: post
permalink: /time-zone-conversion-xamarin-forms-nodatime/
dpsp_networks_shares:
    - 'a:5:{s:8:"facebook";i:18;s:9:"pinterest";i:0;s:6:"reddit";i:0;s:7:"twitter";i:0;s:8:"linkedin";i:0;}'
dpsp_networks_shares_total:
    - '18'
dpsp_networks_shares_last_updated:
    - '1690899576'
image: /assets/img/blog/time-zone-conversion/Standard_World_Time_Zones-e1507747115676.png
categories:
    - Xamarin
tags:
    - NodaTime
    - Xamarin.Forms
---

When developing mobile applications for multiple platforms it can be important to understand how to implement time zones conversion in elegant manner.

Xamarin Forms is a cross-platform natively backed UI toolkit abstraction that allows us to easily create user interfaces that can be shared across Android, iOS, and Windows Phone. It’s implementation of XAML allows us to define user interfaces using markup rather than code, and it is well suited for use with the MVVM (Model-View-ViewModel) application architecture.

First thought on time zone conversions would be to use .NET native TimeZoneInfo class. Since key component of building cross-platform applications is being able to share code across various platform-specific projects, most of the code should be written in Shared projects, Portable Class Libraries (PCLs) or in .NET Standard Libraries ([more info](https://developer.xamarin.com/guides/cross-platform/application_fundamentals/code-sharing/){:target="_blank"}). PCL projects target specific profiles that support a known set of .NET Base Class Library (BCL) classes/features for targeting platforms. .NET Standard is similar to PCL, but with a simpler model for platform support and a greater number of classes from the BCL.

All this implies that only limited set of .NET features is available for PCLs. This is true for TimeZoneInfo class as well, where most of the methods are not available for PCL (including FindSystemTimeZoneById). As nicely explained on [Stackoverflow](https://stackoverflow.com/questions/24176274/net-pcl-exception-while-converting-time-from-utc-to-specified-timezone){:target="_blank"}:

> This is all entirely by design. Timezone conversions requires an operating system with a database that keeps track of the timezone rules across the world. Available on a desktop class machine, not available on limited devices like a phone. Without the database, you can only know something about UTC and the timezone for which the device was configured.

Here is where [NodaTime](http://nodatime.org/){:target="_blank"} library comes handy, powerful date and time library which also has PCL version (library is available on [NuGet](https://www.nuget.org/packages/NodaTime/){:target="_blank"}).

It this example we expect that DateTime data comes from server in UTC format. Pros for storing DateTime in UTC are numerous. You can familiarize yourself with Time Zones best practices [here](http://stackoverflow.com/questions/2532729/daylight-saving-time-and-time-zone-best-practices){:target="_blank"}.

## Converter

First we will implement IValueConverter which will be used from XAML markup. Our UtcToZonedDateTimeConverter should implement Convert and ConvertBack methods.

```csharp
public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
{
	if (value == null || value.GetType() != typeof(DateTime))
	{
		return DateTime.MinValue;
	}

	if (parameter == null || parameter.GetType() != typeof(string) || !DateTimeZoneProviders.Tzdb.Ids.Contains((string)parameter))
	{
		return value;
	}

	var timeZone = DateTimeZoneProviders.Tzdb[(string)parameter];
	var utcDateTime = DateTime.SpecifyKind((DateTime)value, DateTimeKind.Utc);
	var zonedDateTime = Instant.FromDateTimeUtc(utcDateTime).InZone(timeZone).ToDateTimeUnspecified();

	return zonedDateTime;

}
```

Convert method should receive DateTime object as value and TimeZoneId as parameter.  
Method is expecting DateTime object in UTC format (even though its Kind may not be set to UTC explicitly).  
TimeZoneIds should be in [IANA format](http://www.iana.org/time-zones){:target="_blank"}.  
If parameters are correct, we will obtain TimeZone object and use it to convert DateTime to specified time zone.

```csharp
public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
{
	if (value == null || value.GetType() != typeof(DateTime))
	{
		return DateTime.MinValue;
	}

	// If should be set to DateTimeKind.Unspecified in Convert
	if (((DateTime)value).Kind != DateTimeKind.Unspecified)
	{
		return value;
	}

	if (parameter == null || parameter.GetType() != typeof(string) || !DateTimeZoneProviders.Tzdb.Ids.Contains((string)parameter))
	{
		return value;
	}

	var localDateTime = LocalDateTime.FromDateTime((DateTime)value);
	var timeZone = DateTimeZoneProviders.Tzdb[(string)parameter];

	var zonedDbDateTime = timeZone.AtLeniently(localDateTime);
	return zonedDbDateTime.ToDateTimeUtc();

}
```

ConvertBack method in similar manner converts time zoned DateTime back to DateTime in UTC format.

## XAML markup

In XAML file we will load Converter as resource:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://xamarin.com/schemas/2014/forms"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="TimeZoneConversion.MainPage"
             BindingContext="{Binding Source={StaticResource Locator}, Path=Main}" 
             xmlns:converters="clr-namespace:TimeZoneConversion.Converters;assembly=TimeZoneConversion">

  <ContentPage.Resources>
    <ResourceDictionary>

        <converters:UtcToZonedDateTimeConverter x:Key="UtcToZonedDateTimeConverter" />

    </ResourceDictionary>
</ContentPage.Resources>
```

Once converter is available we are ready to use it as static resource to convert bound DateTime property:

```xml
<Label Text="{Binding Time, 
       StringFormat='Time: {0:MM/dd/yyyy hh:mm tt}', 
       Converter={StaticResource UtcToZonedDateTimeConverter},
       ConverterParameter='America/Chicago'}"
       HorizontalOptions="Center" VerticalOptions="Center"></Label>
```

With this kind of implementation we have functional time zone converter which could be easily used in XAML views whenever needed.

Check out working example on my [GitHub repository](https://github.com/markolazic88/TimeZoneConversion){:target="_blank"}.