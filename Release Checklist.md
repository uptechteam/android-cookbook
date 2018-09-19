Welcome to this page. Seems like you have a release coming - congratulations!
Please follow this checklist to make sure you don't forget something important

# Before the release:
1. Verify with the client that the current build is OK to submit
2. Check the version name: should be increased compared to the previous release OR should be the value the client needs (not 0.0.1 instead of 1.0, for example)
3. Install and check the build itself: should be the latest one
4. Run a regression testing for critical features
5. Integrate some crash tracking service: Crashlytics, HockeyApp, etc
6. Integrate any analytics if needed
7. Take into account time for the app’s review by Play Store when planning the release date
8. Make sure the app’s logo looks ok on all devices
9. Prepare the Release Notes
10. Follow git flow to make sure you branches are not messed up
11. Turn off debug logs to avoid security breaches
12. Check that constants for all integrated services are for prod
13. Make sure you’ve build a Prod APK
14. Make sure you wrote down credentials for the keystore somewhere or have them safely hidden and recoverable in your mind palace

# After the release:
1. Pay special attention to the client’s requests during the release time, it’s critical that you react quickly to any issues that occur
2. Monitor the crash tracking service you’ve integrated on step 5 for at least 3 days after the release
