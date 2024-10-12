---
title: "Bottom tab bar navigation and scrolling"
date: 2024-10-01
---
#UIKit, #Tab

Story:
The Pinterest iOS app has a bottom navigation bar containing 5 tabs: Home, Search, Create, Chat, and User.
Within the User view, there is a top navigation bar containing 3 tabs: Pins, Boards, and Collages.

After launching the app, tap User tab navigates to the User view on the Pins top tab's page.

Requirements:
- If user navigate to other subviews by tapping other top navigation tabs (Boards, Collages) and they tap bottom User tab, they should be navigated back to the Pins top tab's view.
- If user is already at the Pin's top tab, but the page has been scrolled through, the page should scroll back up to the beginning of the view.

