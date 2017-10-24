
        **<span class="wysiwyg-underline">Symptoms</span>**

*   App size is bigger than the over-the-air download limit and forces iOS users to download the app via WiFi.
*   The universal binary of the app or single slices are bigger than allowed from the Apple submission guidelines.

**<span class="wysiwyg-underline">Cause</span>**

My app is bigger than the over-the-air download limit on iOS and bigger than the main application bundle size on tvOS. 

The [otool](http://www.unix.com/man-page/osx/1/otool/) report shows that single binary slices (32bit, 64bit) or the universal binary are bigger than the allowed limits from the Apple submission guidelines.

**<span class="wysiwyg-underline">Resolution</span>**

[IL2CPP](http://blogs.unity3d.com/2015/05/06/an-introduction-to-ilcpp-internals/) converts ahead-of-time (AOT) C# code into C++ code which will be compiled into the final binary of your app using XCode and LLVM. This can lead to bigger universal binaries or app slices (32bit,64bit). Your code can be optimised depending on several factors. Read the following to gain insight in what you can do to improve your build size:

<div>

*<span class="wysiwyg-font-size-medium">**Why is the binary size of IL2CPP bigger than with mono?  
**</span>*

</div>
<div>

Two things apply:

</div>

*   If you make a universal build, there is a 32bit slice and a 64bit slice, which contains the exact same executable for 2 different architectures so nearly doubled size.
*   IL2CPP produced bigger builds even if you just compare ARMv7 Mono with ARMv7 IL2CPP; this had to do with how we dealt with the type metadata. It was resolved in Unity 4.6.4p2 and improved further up to 4.7.0f1 and 5.4.0f1.
<div>With "Universal" builds, you will always yield larger binary sizes than just having ARMv7 or ARM64. The removal of the managed assemblies already made the builds smaller (in Unity 4.6.3f1), the real improvement came when we were able to construct at least generic typeinfos at runtime, but this is now partly possible (since Unity 4.6.4p1). Universal builds include 32-bit and 64-bit slices in the fat executable which results in at least doubled size of standard mono 32bit.</div>
<div> </div>
<div>We made improvements to generic type and array generation in Unity 4.6.4p1 too. Apart from that, there's information here about reducing your iOS build size [here](http://docs.unity3d.com/Manual/iphone-playerSizeOptimization.html). For additional information read in ***Stripping*** below.</div>
<div> </div>
<div>Please always make a comparison table with mono (ARMv7), IL2CPP (ARMv7), IL2CPP (ARM64), IL2CPP Universal and which striping settings you use to have all information to compare!</div>
<div> </div>
<div>It is always worth checking the actual binary size, as this is the one affected by the source code compilation referred to as the fat file containing binary from ARMv7 and ARM64 if it is a universal build and therefore nearly doubled size of a single ARMv7 mono or IL2CPP build. If you make a universal build, there is a 32bit slice and a 64bit slice, which contains the exact same executable for 2 different architectures so nearly doubled size. If you want to compare the output directly please compare ARMv7 mono with ARMv7 IL2CPP.</div>
<div>* *</div>
<div>***<u>Which sizes limit does Apple check?</u>***</div>
<div>  
Read the details for app store submission [here](https://developer.apple.com/library/ios/documentation/LanguagesUtilities/Conceptual/iTunesConnect_Guide/Chapters/SubmittingTheApp.html).  </div>
<div>

*   

For apps whose `MinimumOSVersion` is less than 7.0: maximum of 80 MB for the total of all `__TEXT` sections in the binary.

*   

For apps whose `MinimumOSVersion` is 7.x through 8.x: maximum of 60 MB per slice for the `__TEXT` section of each architecture slice in the binary.

*   

For apps whose `MinimumOSVersion` is 9.0 or greater: maximum of 400 MB for the size of the Mach-O binary file.

</div>
<div>Apple has a 80MB limit for the 32bit+64bit code segment in total if you support a minimum OS of less than iOS 7.0. Apps that are set to minimum OS of 7.0 or greater have a binary size limit of 60MB (60 000 000 bytes) per architecture slice. 100MB (mobile data) application limit (user needs to download via Wifi if >100MB) and max 4GB application limit. 80MB and 60MB/60MB is the limit for code segment size included in the fat binary for the 32bit and 64bit slice. You need to use otool to get the right numbers. 100MB applies to to the .ipa size, this determines if users need a WiFi connection to download your app. Follow following steps to get the right numbers:</div>
<div> </div>

1.  Build your app in release mode. Do not use debug mode, as it does not represent your final app. Ensure you use correct optimisation, more information can be found here: [http://docs.unity3d.com/Manual/iphone-playerSizeOptimization.html](http://docs.unity3d.com/Manual/iphone-playerSizeOptimization.html).
2.  Archive your app in XCode and export the app with your deployment certificate to an .ipa.
3.  Use the estimate button to get the estimated size of your app to ensure if you are above or below 100MB or the overall size of 4GB.  
*<u>Note:</u>* since XCode 6.3 there is no estimate button anymore. You can calculate it by using following formula, but be aware that the compression coefficient might vary:  
*app_store_size = sizeof(.ipa/(zip with exec_binary removed)) + sizeof(codesegment(exec_binary)) + 0.2 * sizeof(datasegment(exec_binary))*  
The formula is coming from broader knowledge that only the code segment gets encrypted and the data segment compression ratio can be verified pretty easy.  
Extract the data segment from executable via dd command (you can specify byte offset + length) and then try to compress it. The code segment gets scrambled to look like a perfect noise. You need to descramble it with a decryption key before executing. iTunes/App Store app manages the keys. That's why we add code segment as whole without adjusting for compression ratio.
4.  Once you have created your .ipa (Which should have nearly the same size as the estimated size button in XCode returns) and navigate the Terminal.app to the folder the .ipa was generated in and run otool. 
5.  Use otool to extract the output you need to see if you are close to the the 80MB limit.  

*otool -l <your_app_name>.app/<your_app_name> *or *size <your_app_name>.app/<your_app_name>*  

6.  Now you can retrieve the output depending on the architecture (armv7 + a section arm64 if it's a universal build) Gather the information for armv7 (LC_SEGMENT) and do the same for arm64 if applicable (LC_SEGMENT_64)

    1.  Locate the LC_SEGMENT with segname __TEXT and take the filesize   
code segment size = 30474240 ~= 30MB
    2.  Locate the LC_SEGMENT with segname __DATA and take the filesize   
data segment size (mostly metadata) = 10420224 ~= 10 MB
This results in following otool result table:  

architecture armv7:  
code segment size = 30474240 ~= 30MB  
data segment size (mostly metadata) = 10420224 ~= 10 MB  
segment + data segment = 30 + 10 = 40MB  

architecture arm64:  
code segment size = 25559040 ~= 26MB  
data segment size (mostly metadata) = 17727488 ~= 18 MB  
segment + data segment = 26 + 18 = 44MB  

Apple uses for their check armv7(code segment)+arm64 (code segment) which results in a otool report of 30MB + 26MB = 56MB in this example which is below the 80MB for <7.0 and 30MB & 26MB are each below 60MB >=7.0 for a universal build.
<div>Additionally to these checks, it is easy to make a test app in release mode and submit it to iTunes Connect, the static project checker in the beginning of the upload progress should alert you if a slice is over a certain limit. Once you upload the app to iTunes Connect you can view the expected sizes for Compressed file Size, Download Size and Install Size for all platforms and Universal build. See image below as example (does not correspond to this example number above). </div>
<div> </div>
<div>![](/hc/en-us/article_attachments/203491416/Screen_Shot_2016-04-15_at_18.15.40.png)
<div>

iOS devices using iOS 9.0 and above will download only the slice (32 or 64) depending on their needs. Apple splits the binary and creates a separate package, so those platforms e.g. iPhone 6s should have a smaller Download Size without you doing anything to your game. This is completely unrelated to ODR or Bitcode. Devices running below iOS 9.0 will still need to download the Universal build however.

</div>
</div>
<div><span class="wysiwyg-font-size-medium">***Stripping******  
***</span></div>
<div> </div>
<div>The IL2CPP scripting backend always does byte code stripping, no matter what the *Stripping Level* setting is in the editor. The best option for a user is to set the *Stripping Level* option in the editor to *Disabled*, as the affect of any other *Stripping Level* option will likely be minimal on executable size, because IL2CPP will strip anyway. If you choose a *Stripping Level* value other than *Disabled* you could run into problems, because then the IL2CPP build toolchain in the Editor will attempt to determine which native engine code is used by your scripting assemblies, and only register that native code when the player starts. If you encounter problems with your stripping settings and you believe they are wrong, please submit a bug report.</div>
<div> </div>
<div>***5.3.x build size increase due to Bitcode***</div>
<div> </div>
<div>When building iOS applications with Unity 5.3.x there is a increase of the build size. This is due to Bitcode support which was enabled in 5.3.x. Why this is good and how to deal with it can be read [here](/hc/en-us/articles/209933103).  </div>
<div> </div>
<div>***Analysing scripts with MapFileParser** *</div>
<div>

You can analyse scripts even more and deeply investigate script contributions to build size. You can inspect the executable size and the contributions that scripts have with the **MapFileParser** utility to get some information. The tool processes the map file generated by Xcode when you build the app (it won't be generated when archiving the app). The *-stats* flag is not implemented in 5.2.4p1 and below. You can find the MapFileParser in the Unity.app directory or in your generated XCode project root directory. The your_map_file.map should be found in the Derived Data folder of the Xcode linker output.

![](/hc/en-us/article_attachments/203780483/Screen_Shot_2016-04-22_at_12.00.57.png)

The full path would be similar to Library/Developer/Xcode/DerivedData/Unity-iPhone-glmxdxebssyebsfcbtobeuasetge/Build/Intermediates/Unity-iPhone.build/<mode>-iphoneos/Unity-iPhone.build/alpin-LinkMap-normal-<architecture>.txt 

You can analyse the map file using following command in the Terminal.app 

*<Contents/Tools/MapFileParser/>MapFileParser -format=Clang <your_map_file.map> -stats*

Analysing the names of the functions is the best way to make the distinction between different code (user scripts, static libraries). Code generated from the managed assemblies will show up with _m suffixes e.g:

*_TestScript_Start_m4164746442: 10 bytes  
_TestScript_Update_m263972995: 10 bytes  
_TestScript*_ctor_m922641354: 24 bytes

This might help you to understand your code, plugins or engine code even better.

</div>
<div class="article-body markdown">**<span class="wysiwyg-underline">More Information</span>**</div>
<div class="article-body markdown">

*   Read more about [Optimizing iOS & tvOS apps with app thinning and IL2CPP](/hc/en-us/articles/208411836)
*   Read more about IL2CPP [here](http://blogs.unity3d.com/2015/05/06/an-introduction-to-ilcpp-internals/)
*   <div>Read more about IL2CPP in the [IL2CPP blog posts](http://blogs.unity3d.com/?s=il2cpp)</div>

*   <div>Read more about [iOS 64bit support  
](http://blogs.unity3d.com/?s=ios+64)</div>

*   Read move about [IL2CPP Build Size improvement](http://forum.unity3d.com/threads/il2cpp-build-size-improvements.322079/)
</div>

This article applies to Unity versions 5.2.0p1 and higher, XCode 7 and higher, iOS 9.0 and higher

      

      










<div class="line number1 index0 alt2">

<span class="wysiwyg-underline wysiwyg-font-size-medium">**Symptoms**</span>

</div>
<div class="line number3 index2 alt2">

*   When I export a package using *Assets -> Export package*, it exports everything.

</div>
<div class="line number4 index3 alt1">

<span class="wysiwyg-underline">**Cause**</span>

</div>
<div class="line number4 index3 alt1"> </div>
<div class="line number4 index3 alt1">

When exporting packages that include scripts, it includes all scripts that are in the project. This happens because there is no way to track dependencies between scripts. If you include dependencies and one of the dependencies is a script, then you need all scripts.

</div>
<div class="line number4 index3 alt1"> </div>
<div class="line number4 index3 alt1">

<span class="wysiwyg-underline">**Resolution**</span>

</div>
<div class="line number4 index3 alt1">

You can use the function [AssetDatabase.ExportPackage](http://docs.unity3d.com/ScriptReference/AssetDatabase.ExportPackage.html) and multiple [ExportPackageOptions](http://docs.unity3d.com/ScriptReference/ExportPackageOptions.html). Please be aware that this workaround could cause some errors with missing scripts. *Assets -> Export package* is the suggested and secure option.

</div>
<div class="line number4 index3 alt1">

<span class="wysiwyg-underline">**More Information**</span>

</div>
<div class="line number4 index3 alt1"> </div>
<div class="line number4 index3 alt1">

[http://docs.unity3d.com/ScriptReference/AssetDatabase.ExportPackage.html](http://docs.unity3d.com/ScriptReference/AssetDatabase.ExportPackage.html)

</div>
<div class="line number4 index3 alt1">

[http://docs.unity3d.com/ScriptReference/ExportPackageOptions.html](http://docs.unity3d.com/ScriptReference/ExportPackageOptions.html)

</div>
<div class="line number4 index3 alt1"> </div>
      
	
	





# Angular JumpStart with TypeScript

The goal of this jumpstart app is to provide
a simple way to get started with Angular 2+ while also showing several key Angular features. The sample
relies on System.js to load TypeScript modules and the required scripts used in the application.

## Angular Concepts Covered

* TypeScript version that relies on classes and modules
* Modules are loaded with System.js
* Defining routes including child routes and lazy loaded routes
* Using Custom Components including custom input and output properties
* Using Custom Directives
* Using Custom Pipes
* Defining Properties and Using Events in Components/Directives
* Using the Http object for Ajax calls along with RxJS observables
* Working with Utility and Service classes (such as for sorting and Ajax calls)
* Using Angular databinding Syntax [], () and [()]
* Using template-driven and reactive forms functionality for capturing and validating data
* Optional: Webpack functionality is available for module loading and more (see below for details)
* Optional: Ahead-of-Time (AOT) functionality is available for a production build of the project (see below for details)

## Running the Application

1. Install `Node.js 6.5` or higher. *IMPORTANT: The server uses ES2015 features so you need Node 6.x or higher!!!!*

1. Run `npm install` to install app dependencies

1. Run `npm start` in a separate terminal window to build the TypeScript, watch for changes and launch the web server

1. Go to http://localhost:3000 in your browser

Looking for expert onsite Angular/TypeScript training? We've trained the biggest (and smallest :-)) companies around the world for over 15 years. For more information visit http://codewithdan.com. 

Simply clone the project or download and extract the .zip to get started. Here are a few
screenshots from the app:

<img width="500" src="src/images/screenshots/cards.png" border="0" />

<br /><br />

<img width="500" src="src/images/screenshots/grid.png" border="0" />

<br /><br />

<img width="500" src="src/images/screenshots/orders.png" border="0" />

<br /><br />

<img width="500" src="src/images/screenshots/details.png" border="0" />

## Running the Application using WebPack (and optionally AOT)

The Angular JumpStart application uses System.JS by default for module loading mainly to keep the app
as simple as possible and focused on Angular concepts rather than on a ton of configuration. However, Webpack 
can also be used to handle working with modules plus perform tasks such as bundling, minification, conversion
of TypeScript to JavaScript, start up a dev web server and much more. 

If you'd like to use WebPack instead of SystemJS you'll need to modify a few things in the application. Here's a
list of the required steps to get the application going using Webpack:

1. Do a global search and replace in the project to comment out all references to `moduleId` in each component since it isn't used by Webpack:		
    *Find:*             `moduleId: module.id,`		
	
    *Replace with:*     `//moduleId: module.id,`		
	
If you plan on only using Webpack and not going back to SystemJS you can completely remove `moduleId: module.id,` if you'd like.

1. Open `src/app/app-routing.module.ts` and change `app/` to `./` for all `loadChildren` paths. For example:

    *Change:*     loadChildren: 'app/customers/customers.module#CustomersModule'

    *To:*         loadChildren: './customers/customers.module#CustomersModule'

1. Run `npm install` to install app dependencies

1. Optional (for AOT). If you'd like to run the application using Ahead of Time (AOT) compilation, 
   run the following command to set the NODE_ENV environment variable to production and create an AOT build:

    *Mac*:      `export NODE_ENV=production`
    *Windows:   `set NODE_ENV=production`

1. Run `npm run build` in a console window. This will generate the required script assets needed to run the application
   and place them in the `src/devDist` folder for a development build or `src/dist` for a production/AOT build (if you performed the previous step). 
   It will also watch for any code changes that are made and rebuild the script bundles as needed.

1. Run `node server.js` in another command window to start the server.

1. Navigate to http://localhost:3000 in a browser.


