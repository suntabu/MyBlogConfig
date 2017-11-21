

From https://unity3d.com/cn/learn/tutorials/topics/best-practices/optimizing-ui-controls?playlist=30089

#### UI text
**When adding text to a UI, always remember that the text glyphs are actually rendered as individual quads, one per character.**

Use [TextMesh Pro](https://www.assetstore.unity3d.com/en/#!/content/84126) instead, It's free now.

#### Scroll Views

Two basic approaches to populating a scrollview:
- Fill it with all of the elements necessary to represent all of the scroll view’s content
- Pool the elements, repositioning them as needed to represent visible content.
