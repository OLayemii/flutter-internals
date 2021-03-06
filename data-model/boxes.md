# Boxes

## What are the render box building blocks?

* `RenderBox` models a two dimensional box with a width, height, and position \(`RenderBox.size.width`, `RenderBox.size.height`,`RenderBox.parentData.offset`\). The box's top-left corner defines its origin, with the bottom-right corner corresponding to `(width, height)`.
* `BoxParentData` stores the child's offset in the parent’s coordinate space \(`BoxParentData.offset`\). By convention, this data may not be accessed by the child.
* `BoxConstraints` describes immutable constraints expressed as a maximum and minimum width and height ranging from zero to infinity, inclusive. Constraints are satisfied by concrete sizes that fall within this range.
  * Box constraints are classified in several ways:
    * `BoxConstraints.isNormal`: minimum is greater than zero and less than or equal to the maximum in both dimensions.
    * `BoxConstraints.tight, BoxConstraints.isTight`: minimum and maximum values are equal in both dimensions.
    * `BoxConstraints.loose`: minimum value is zero, even if maximum is also zero \(i.e., loose and tight\).
    * `BoxConstraints.hasBoundedWidth`, `BoxConstraints.hasBoundedHeight`: the corresponding maximum is not infinite.
      * Constraints are unbounded when the maximum is infinite.
    * `BoxConstraints.expanding`:  both maximum and minimum values are infinite in the same dimension \(i.e., tightly infinite\).
      * Expanding constraints imply that the corresponding dimension will be determined by other incoming constraints \(e.g., established by containing UI\). Dimensions must ultimately be finite.
    * `BoxConstraints.hasInfiniteWidth`, `BoxConstraints.hasInfiniteHeight`: the corresponding minimum is infinite \(thus, the maximum must also be infinite; this is the same as expanding\).
  * Box constraints can be evaluated \(`BoxConstraints.isSatisfiedBy`\), applied to a `Size` \(`BoxConstraints.constrain`\), tightened relative to constraints \(`BoxConstraints.tighten`\), loosened by setting minimums to zero \(`BoxConstraints.loosen`\), and intersected \(`BoxConstraints.enforce`\). Constraints can also be scaled standard algebraic operators.
* `BoxHitTestResult` is a `HitTestResult` subclass that captures each `RenderBox` \(a`HitTestTarget`\) that reported being hit in order of decreasing precedence.
  * Instances include box-specific helpers intended to help transform global coordinates to local coordinates \(e.g., `BoxHitTestResult.addWithPaintOffset`, `BoxHitTestResult.addWithPaintTransform`\).
* `BoxHitTestEntry` represents a box that was hit during hit testing. It captures the position of the collision in local coordinates \(`BoxHitTestEntry.localPosition`\).

## How do boxes model children?.

* `ContainerBoxParentData` extends `BoxParentData` with `ContainerParentDataMixin`. This combines a child offset \(`BoxParentData.offset`\) with next and previous pointers \(`ContainerParentData.previousSibling`, `ContainerParentData.nextSibling`\) to describe a doubly linked list of children.
* `RenderObjectWithChildMixin` and `ContainerRenderObjectMixin` can both be used with a type argument of `RenderBox`. The `ContainerRenderObjectMixin` accepts a parent data type argument; `ContainerBoxParentData` is compatible and adds support for box children.
* `RenderBoxContainerDefaultsMixin` adds useful defaults to `ContainerRenderObjectMixin` for render boxes with children. Type constraints require that children extend `RenderBox` and parent data extends `ContainerBoxParentData`.
  * This mixin provides support for hit testing \(`RenderBoxContainerDefaultsMixin.defaultHitTestChildren`\), painting \(`RenderBoxContainerDefaultsMixin.defaultPaint`\), and listing \(`RenderBoxContainerDefaultsMixin.getChildrenAsList`\) children.
* `RenderProxyBox` delegates all methods to a single `RenderBox` child, adopting the child's size as its own size and positioning the child at its origin. Proxies are convenient for writing subclasses that selectively override some, but not all, of a box's behavior. This box's implementation is provided by `RenderProxyBoxMixin`, which provides an alternative to direct inheritance. Proxy boxes uses `ParentData` rather than `BoxParentData` since the child's offset is never used.
  * There are numerous examples throughout the framework:
    * `RenderAbsorbPointer` overrides `RenderProxyBox.hitTest` to disable hit testing for a subtree \(i.e., by not testing its child\) when `RenderAbsorbPointer.absorbing` is enabled.
    * `RenderAnimatedOpacity` listens to an animation \(`RenderAnimatedOpacity.opacity`\) to render its child with varying opacity \(via `RenderAnimatedOpacity.paint`\). This box manages its listener by overriding `RenderProxyBox.attach` and `RenderProxyBox.detach`. It selective inserts an `OpacityLayer` \(i.e., for translucent values only\), and manages `RenderBox.alwaysNeedsCompositing` to reflect whether a layer will be added \(via `RenderAnimatedOpacityMixin._updateOpacity`, called in response to the animation\).
    * `RenderIntrinsicWidth` overrides `RenderProxyBox.performLayout` to adopt a size that corresponds to its child's intrinsic width, subject to the incoming constraints. It also overrides intrinsic sizing methods since it also provides size snapping \(`RenderBox.stepWidth`\).
* `RenderShiftedBox` delegates all methods to a single `RenderBox` child, but leaves layout undefined. Subclasses override `RenderShiftedBox.performLayout` to assign a position to the child \(via `BoxParentData.offset`\). Otherwise, this subclass is analogous to `RenderProxyBox`.
  * `RenderPadding` overrides `RenderShiftedBox.performLayout` to deflate the incoming constraints by the resolved padding amount. The child is laid out using the new, padded constraints and positioned within the padded region.
  * `RenderBaseline` overrides `RenderShiftedBox.performLayout` to align its child's baseline \(via `RenderBox.getDistanceToBaseline`\) with an offset from its top edge \(`RenderBaseline.baseline`\). It then sizes itself such that its bottom is coincident with the child's baseline \(this can potentially truncate the top of the child\).

## How do render boxes handle layout?

* Boxes use the standard `RenderObject` layout protocol to map from incoming `BoxConstraints` to a concrete `Size` \(stored in `RenderBox.size`\). Layout also determines the child's offset relative to the parent \(`RenderBox.parentData.offset`\). This information should not be read by the child during layout.
* Boxes add support for intrinsic dimensions \(an ideal size, computed outside of the standard layout protocol\) as well as baselines \(a line to use for vertical alignment, typically used when laying out text\). `RenderBox` instances track changes to these values and whether the parent has queried them; if so, when that box is marked as needing layout, the parent is marked as well.
  * Intrinsic dimensions are cached \(`RenderBox._cachedIntrinsicDimensions`\) whenever they're computed \(via `RenderBox._computeIntrinsicDimension`\). The cache is disabled during debugging.
  * Baselines are cached \(`RenderBox._cachedBaselines`\) whenever they're computed \(via `RenderBox.getDistanceToActualBaseline`\). Different baselines are computed for alphabetic and ideographic text.
* `RenderBox.markNeedsLayout` marks the parent as needing layout if the box's intrinsic dimension or baseline caches have been modified \(i.e., this implies that the parent has accessed the box's "out-of-band" geometry\). If so, both are cleared so that new values are computed after the next layout pass.
* By default, boxes that are sized by their parents adopt the smallest size permitted by incoming constraints \(via `RenderBox.performResize`\).
* Boxes can have non-box children. In this case, the constraints provided to children will need to be adapted from `BoxConstraints` to the appropriate type.

## How do render boxes handle painting?

* Boxes use the standard `RenderObject` painting protocol to paint themselves to a provided canvas. The canvas's origin isn't necessarily coincident with the box's origin; the offset provided to `RenderBox.paint` describes where the box's origin falls on the canvas. The canvas and the box will always be axis-aligned.
* `RenderBox.paintBounds` describes the region that will be painted by a box. This determines the size of the buffer used for painting and is expressed in local coordinates. It need not match `RenderBox.size`.
* If the render box applies a transform when painting \(e.g., painting at a different offset than the one provided\), `RenderBox.applyPaintTransform` must apply the same transformation to the provided matrix.
  * `RenderBox.globalToLocal` and `RenderBox.localToGlobal` rely on this transformation to map from global coordinates to box coordinates and vice versa.
  * By default, `RenderBox.applyPaintTransform` applies the child’s offset \(via `child.parentData.offset`\) as a translation.

## How do render boxes handle hit testing?

* Boxes support hit testing, a subscription-like mechanism for delegating and processing events. The box protocol implements a custom flow rather than extending the`HitTestable` interface \(though `RendererBinding`, the hit testing entry point, does implement this interface\). 
  * `RendererBinding.hitTest` is invoked using mixin chaining \(via `GestureBinding._handlePointerEvent`\).  The binding delegates to `RenderView.hitTest` which tests its child \(via `RenderBox.hitTest`\).
  * `RenderBox.hitTest` determines whether an offset \(in local coordinates\) falls within its bounds. If so, each child is tested in sequence \(via`RenderBox.hitTestChildren`\) before the box tests itself \(via `RenderBox.hitTestSelf`\). By default, both methods return `false` \(i.e., boxes do not handle events or forward events to their children\).
  * Boxes subscribe to the related event stream \(i.e., receive `RenderBox.handleEvent` calls for events related to the interaction\) by adding themselves to `BoxHitTestResult`. Boxes added earlier take precedence.
  * All boxes in the `BoxHitTestResult` are notified of events \(via `RenderBox.handleEvent`\) in the order that they were added.

## What are intrinsic dimensions?

* Conceptually, the intrinsic dimensions of a box are its natural dimensions \(i.e., the size it “wants” to be\). The precise definition depends on the box's implementation and semantics \(i.e., what the box represents\). 
  * Intrinsic dimensions are often defined in terms of the intrinsic dimensions of children and are therefore expensive to calculate \(typically traversing an entire subtree\).
* Intrinsic dimensions often differ from the dimensions produced by layout \(except when using the `IntrinsicHeight` and `IntrinsicWidth` widgets, which attempt to layout a child using its intrinsic dimensions\).
  * Intrinsic dimensions are generally ignored unless one of these widgets are used \(or another widget that explicitly incorporates intrinsic dimensions into its own layout, e.g., `RenderTable`\).
* The box model describes intrinsic dimensions in terms of minimum and maximum values for width and height \(via `RenderBox.computeMinIntrinsicHeight`, `RenderBox.computeMaxIntrinsicHeight`, etc.\). Both receive a value for the opposite dimension \(if infinite, the other dimension is unconstrained\); this is useful for boxes that define one intrinsic dimension in terms of the other \(e.g., text\).
  * Minimum intrinsic width is the smallest width before the box cannot paint correctly without clipping.
    * Intuition: making the box thinner would clip its contents.
    * If width is determined by height according to the box's semantics, the incoming height \(which may be infinite, i.e., unconstrained\) should be used. Otherwise, ignore the height.
  * Minimum intrinsic height is the same concept for height.
  * Maximum intrinsic width is the smallest width such that further expansion would not reduce minimum intrinsic height \(for that width\).
    * Intuition: making the box wider won’t help fit more content.
    * If width is determined by height according to the box's semantics, the incoming height \(which may be infinite, i.e., unconstrained\) should be used. Otherwise, ignore the height.
  * Maximum intrinsic height is the same concept for height.
* The specific meaning of intrinsic dimensions depends on the box's semantics.
  * Text is width-in-height-out.
    * Maximum intrinsic width: the width of the string without line breaks \(increasing the width would not shrink the preferred height\).
    * Minimum intrinsic width: the width of the widest word \(decreasing the width would clip the word or cause an invalid break\).
    * Intrinsic heights are computed by laying out text with the provided width.
  * Viewports ignore incoming constraints and aggregate child dimensions without clipping \(i.e., ideally, a viewport can render all of its children without clipping\).
  * Aspect ratio boxes use the incoming dimension to compute the queried dimension \(i.e., width to determine height and vice versa\). If the incoming dimension is unbounded, the child's intrinsic dimensions are used instead.
  * When intrinsic dimensions cannot be computed or are too expensive, return zero.

## What are baselines?

* Baselines are a concept borrowed from text rendering to describe the line upon which all glyphs are placed. Portions of the glyph typically extend below this line \(e.g., descenders\) as it serves primarily to vertically align a sequence of glyphs in a visually pleasing way. Characters from different fonts can be visually aligned by positioning each span of text such that baselines are collinear.
  * Boxes that define a visual baseline can also be aligned in this way.
* Boxes may specify a baseline by implementing `RenderBox.computeDistanceToActualBaseline`. The returned value represents a vertical offset from the top of the box. Values are cached until the box is marked as needing layout.
  * Boxes typically return null \(i.e., they don't define a logical baseline\), a value intrinsic to what they represent \(i.e., the baseline of a span of text\), delegate to a single child, or use `RenderBoxContainerDefaultsMixin` to produce a baseline from a set of children. 
    * `RenderBoxContainerDefaultsMixin.defaultComputeDistanceToFirstActualBaseline` returns the first valid baseline reported by the set of children, adjusted to account for the child's offset.
    * RenderBoxContainerDefaultsMixin.defaultComputeDistanceToHighestActualBaseline returns the minimum baseline \(i.e., vertical offset\) amongst all children, adjusted to account for the child's offset.
* `RenderBox.getDistanceToBaseline` returns the offset to the box's bottom edge \(`RenderBox.size.height`\) if an actual baseline isn't available \(i.e., `RenderBox.computeDistanceToActualBaseline` returns `null`\).
  * The baseline may only be queried by a box's parent and only after the box has been laid out \(typically during parent layout or painting\).

