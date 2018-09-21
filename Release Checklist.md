Seems like you have a release coming - congratulations!
Please follow this checklist to make sure you don't forget something important.

# Before the release:
1. Verify with the client that the current build is ready to submit.
2. Check the version name: should be increased compared to the previous release OR should be the value the client needs (not 0.1 instead of 1.0, for example).
3. Install and check the build itself: should be the latest one.
4. Run a regression testing for critical features.
5. Integrate some crash tracking service: Crashlytics, HockeyApp, etc.
6. Integrate any analytics if needed.
7. Take into account time for the app’s review by Play Store when planning the release date.
8. Make sure the app’s logo looks ok on all devices.
9. Prepare the Release Notes.
10. Follow git flow to make sure you branches are not messed up.
11. Turn off debug logs to avoid security breaches.
12. Make sure no debug tools (like Stetho, Chuck, etc) are enabled in the release build.
13. Check that constants for all integrated services are for Production.
14. Make sure you’ve built a Production APK.
15. Make sure you wrote down credentials for the keystore in a safe place.
16. Consider starting with a staged roll-out to do a rollback in case any problems occur.
17. If it's an incremental release (an update to the existing app) check for backward compatibility with the previous version: install the previous APK and then on top of it the new APK. An example of things to check is a DB migration.
