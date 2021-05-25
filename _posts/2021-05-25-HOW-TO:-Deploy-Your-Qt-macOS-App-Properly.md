---
author_profile: true
comments: true
share: true
related: true
show_date: true
---
# Introduction:
After a while of searching the internet, and forums, I was unable to find a definitive guide on deploying a Qt app from A to Z on macOS. Given that Apple has a weird code signing and app notarization system. This quick "how-to" will cover how to deploy your Qt macOS, building and linking will not be covered obviously as that differs per project. Hope this helps :)

I have automated it all with a simple python script which will be posted at the bottom, but I will go through it here.

Please note, this assumes you have a valid developer account with Apple.

# Signing and Packaging your Application:
First off, we need to deploy all the shared libraries, sign and package the app.
Before hand, you need to tell Cmake to package your executable into a .app. To do so simply do the following:

```cmake
set(BUNDLE )
if(APPLE)
    set(BUNDLE MACOSX_BUNDLE)
endif()

add_executable(Student_Management_System ${BUNDLE} main.cpp)
```

Now that you can compile and get your .app, you need to deploy all the Qt dependencies, and other dependencies into the bundle. What's wonderful is Qt does this for you :), using macdeployqt! Like so:

`macdeployqt {app_name}.app -always-overwrite`

Then we would need to code sign, using the codesign command, like so:

`codesign -f --deep -v --options runtime -s 'Developer ID Application: {developer_ID}' {app_name}.app`

After all of that, you need to bundle the app as a dmg for notarization, like so:

`hdiutil create /tmp/tmp.dmg -ov -volname '{app_name}' -fs HFS+ -srcfolder '{app_name}.app`

`hdiutil convert /tmp/tmp.dmg -format UDZO -o {app_name}.dmg`

Then we need to sign the dmg.

`codesign -f --deep -v --options runtime -s 'Developer ID Application: {developer_ID}' {app_name}.dmg`


# Notarizing the Application:
After all of that code signing, and bundling the app into a dmg, we need to upload the dmg to Apple for notarization. 

`xcrun altool --notarize-app --primary-bundle-id '{bundle_ID}' --username '{apple_username}' --password '{apple_password}' --file {app_name}.dmg`

The following command, if successful should output the following
```
No errors uploading 'app_name.dmg'.
RequestUUID = 664647b4-adf8-4ac6-971b-XXXXXXX
```
What we need to do is grab that UUID and wait for it to be approved, to check on its status, we can use this command.

`xcrun altool --notarization-info {UUID} -u {apple_username} -p {apple_password}`

When the application has been successfully notarized, we get the following output:
```
No errors getting notarization info.

          Date: 2021-05-25 18:26:08 +0000
          Hash: 986e0c5019XXXXXXXX
    LogFileURL: https://osxapps-ssl.itunes.apple.com/......
   RequestUUID: 664647b4-adf8-4ac6-971b-XXXXXXXXX
        Status: success
   Status Code: 0
Status Message: Package Approved

```

When it has been approved what we finally need to do, is staple the notarization info to the dmg, and validate it.

`xcrun stapler staple {app_name}.dmg`

We should get this output: `The staple and validate action worked!`


# Automating this Process

Congrats! You now have successfully signed and notarized your Qt application. Lets automate this whole thing! Below is a simple, yet messy python script to automate this whole process. Enjoy :)

```python
def qt_deploy_sign_notarize(app_name: str, developer_ID: str,keychain_password: str, apple_username: str,apple_password: str, bundle_ID: str) -> bool:

    # Unlock keychain, incase it is locked. Needed for signing.
    os.system(f"security unlock-keychain -p {keychain_password} login.keychain")

    # Use macdeployqt to deploy or shared libs and dependencies into the app bundle
    os.system(f"~/Qt/6.0.0/clang_64/bin/macdeployqt {app_name}.app -always-overwrite")

    # Sign the app bundle using codesign
    os.system(f"codesign -f --deep -v --options runtime -s 'Developer ID Application: {developer_ID}' {app_name}.app")

    # Create a dmg of the app
    os.system(f"hdiutil create /tmp/tmp.dmg -ov -volname '{app_name}' -fs HFS+ -srcfolder '{app_name}.app'")
    os.system(f"hdiutil convert /tmp/tmp.dmg -format UDZO -o {app_name}.dmg")

    # Sign the dmg
    os.system(f"codesign -f --deep -v --options runtime -s 'Developer ID Application: {developer_ID}' {app_name}.dmg")

    # Notrize app and get the UUID:
    altool_res = subprocess.check_output(f"xcrun altool --notarize-app --primary-bundle-id '{bundle_ID}' --username '{apple_username}' --password '{apple_password}' --file {app_name}.dmg",stderr=subprocess.STDOUT,shell=True).decode("utf-8")
    UUID = altool_res.split("RequestUUID = ")[1].strip()
    print(altool_res)

    while True:
        time.sleep(10)
        try:
            notarization_history_res = subprocess.check_output(f"xcrun altool --notarization-info {UUID} -u {apple_username} -p {apple_password}",stderr=subprocess.STDOUT,shell=True).decode("utf-8")
            if "Package Invalid" in notarization_history_res and "in progress" not in notarization_history_res:
                print(notarization_history_res)
                print("ERROR: Package not notarized..")
                return False
            elif "Package Approved" in notarization_history_res and "in progress" not in notarization_history_res:
                print(notarization_history_res)
                print("Package notarized!")
                print("Stapling....")
                time.sleep(15)
                os.system(f"xcrun stapler staple {app_name}.dmg")
                os.system(f"xcrun stapler validate {app_name}.dmg")
                return True
        except:
            print("ERROR with notarization_history_res")
```