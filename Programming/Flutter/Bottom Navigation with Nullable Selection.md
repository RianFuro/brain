# Goal
The Material bottom navigation bar is nice, but assumes that one item must be selected, by index. This is not ideal in some scenarios, where a bottom navigation should provide quick access to the most used features, but other views are still available.
With this in mind, I wanted a bottom navigation where an item could be selected, but didn't have to be. Based on this goal, a few additional design decisions are extrapolated:
- With an optional selection, only a controlled selection makes sense, instead of ephemeral state. This means the selected item must be provided by the parent.
- Since additional views must exist for this solution to make sense, selecting the current item by index is meaningless. Instead we give each item a key and select an item by that key.
- The solution needs to feel as close to the original as possible, we don't want to sacrefice quality.

# Implementation
The minimal definition for a navigation item looks like this:
```dart
class BottomAppBarItem {  
  const BottomAppBarItem({  
    required this.key,  
    required this.text,  
    required this.icon  
  });  
  final ValueKey key;  
  final IconData icon;  
  final String text;  
}
```
We will use the key to select an item and the icon and text for displaying purposes.

TODO