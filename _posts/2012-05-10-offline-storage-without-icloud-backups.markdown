---
layout: post
title: "Offline storage without iCloud backups"
date: 2012-05-10 14:09
categories: iOS, iCloud
---
Today I got my first App-rejection{% fn_ref 1%}. NRC Media's [In beeld](http://www.nrc.nl/inbeeldappstore) app recently got a major update, which as it turns out didn't follow Apple's [iOS Data Storage Guidelines](https://developer.apple.com/icloud/documentation/data-storage/). However, Apple's review team detected this in the minor bugfix update that I submitted after that. The problem was that all photo's downloaded by the app were backed up into iCloud resulting in an increase of backup size of about 100MB in some cases.

While you can use the `~/Library/Caches` directory to store files that can be reproduced (downloaded again or regenerated), iOS can purge this data at any time, crippling offline support for your app. Marco Arment has a nice write-up about the [consequences of this behaviour](http://www.marco.org/2011/10/13/ios5-caches-cleaning). While his post now contains an update mentioning the change Apple made in iOS 5.0.1 to address this problem, I couldn't find a post about the way you should implement it, so here it is.
<!-- more -->
In iOS 5.0.1 Apple introduced a new file attribute allowing developers to specify which files shouldn't be backed up. They defined four 'Data Handling Categories' described in [Technical Q&A QA1719](http://developer.apple.com/library/ios/#qa/qa1719/_index.html): Critical Data, Cached Data, Temporary Data and Offline Data. Offline Data is defined as "[…] data that can be downloaded or otherwise recreated, but that the user expects to be reliably available when offline".

Offline data should be stored in the `~/Documents` directory or in a subdirectory of `~/Libary` such as `~/Library/Private Documents` and marked with a special file attribute that specifies that the file should not be backupped, but isn't purged when storage runs low. How this attribute should be set is different for iOS version 5.0.1 and 5.1.

In iOS 5.0.1 the attribute named `com.apple.MobileBackup` should be given the value `1` using [`setxattr`](https://developer.apple.com/library/mac/#documentation/Darwin/Reference/Manpages/man2/setxattr.2.html) for every filepath that should not be backed up. This attribute is ignored by iOS version 5.0 an lower, so it can be safely added in any iOS version. Sample code for this is provided by Apple:

{% codeblock Setting the Extended Attribute in iOS 5.0.1 lang:c http://developer.apple.com/library/ios/#qa/qa1719/_index.html Technical Q&A QA1719 %}
#include <sys/xattr.h>
- (BOOL)addSkipBackupAttributeToItemAtURL:(NSURL *)URL
{
    const char* filePath = [[URL path] fileSystemRepresentation];
 
    const char* attrName = "com.apple.MobileBackup";
    u_int8_t attrValue = 1;
 
    int result = setxattr(filePath, attrName, &attrValue, sizeof(attrValue), 0, 0);
    return result == 0;
}
{% endcodeblock %}

In iOS 5.1 Apple added the [`NSURLIsExcludedFromBackupKey`](http://developer.apple.com/library/ios/#DOCUMENTATION/Cocoa/Reference/Foundation/Classes/NSURL_Class/Reference/Reference.html#//apple_ref/doc/c_ref/NSURLIsExcludedFromBackupKey) that can be added to a file URL with the [`setResourceValue:forKey:error:`](http://developer.apple.com/library/ios/#DOCUMENTATION/Cocoa/Reference/Foundation/Classes/NSURL_Class/Reference/Reference.html#//apple_ref/occ/instm/NSURL/setResourceValue:forKey:error:) instance method of `NSURL`. 

These two methods are combined nicely in the following gist:

{% gist 2002382 NSFileManager%2BDoNotBackup.m %}

There is just one caveat: because of a [bug](http://www.openradar.me/radar?id=1597401) in the iOS 5.0 simulator, `&NSURLIsExcludedFromBackupKey == nil` will return false even if that symbol is not defined. This will cause this piece of code to crash in the iOS 5.0 simulator, but it will run fine in the iOS 5.1 simulator and on any iOS device.

- - -

{% footnotes %}
 {% fn Well, that is not entirely true. My first iOS app, an iPhone client <a href='http://fetishwijzer.nl'>Fetishwijzer</a> got rejected and never made it to the App Store, but I kind of expected that. %}
{% endfootnotes %}