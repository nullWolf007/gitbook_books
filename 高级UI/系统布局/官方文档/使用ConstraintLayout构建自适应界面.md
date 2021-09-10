[TOC]

# 使用 ConstraintLayout 构建自适应界面 

* [`ConstraintLayout`](https://developer.android.com/reference/androidx/constraintlayout/widget/ConstraintLayout?hl=zh_cn) 可让您使用扁平视图层次结构（无嵌套视图组）创建复杂的大型布局。它与 [`RelativeLayout`](https://developer.android.com/reference/android/widget/RelativeLayout?hl=zh_cn) 相似，其中所有的视图均根据同级视图与父布局之间的关系进行布局，但其灵活性要高于 `RelativeLayout`，并且更易于与 Android Studio 的布局编辑器配合使用。

* `ConstraintLayout` 的所有功能均可直接通过布局编辑器的可视化工具来使用，因为布局 API 和布局编辑器是专为彼此构建的。 因此，您完全可以使用 `ConstraintLayout` 通过拖放的形式（而非修改 XML）来构建布局。

* 本文提供了使用 [`ConstraintLayout`](https://developer.android.com/reference/androidx/constraintlayout/widget/ConstraintLayout?hl=zh_cn) 在 Android Studio 3.0 或更高版本中构建布局的指南。如果您想了解关于布局编辑器的更多信息，请参阅 Android Studio 指南以[使用布局编辑器构建界面](https://developer.android.com/studio/write/layout-editor?hl=zh_cn)。

* 要查看您能够使用 `ConstraintLayout` 创建的各种布局，请查看 [GitHub 上的约束布局示例项目](https://github.com/android/views-widgets-samples/tree/master/ConstraintLayoutExamples)。

## 一、约束条件概览

* 要在 `ConstraintLayout` 中定义某个视图的位置，您必须为该视图添加至少一个水平约束条件和一个垂直约束条件。每个约束条件均表示与其他视图、父布局或隐形引导线之间连接或对齐方式。每个约束条件均定义了视图在竖轴或者横轴上的位置；因此每个视图在每个轴上都必须至少有一个约束条件，但通常情况下会需要更多约束条件。

* 当您将视图拖放到布局编辑器中时，即使没有任何约束条件，它也会停留在您放置的位置。不过，这只是为了便于修改；当您在设备上运行布局时，如果视图没有任何约束条件，则会在位置 [0,0]（左上角）处进行绘制。

* 在图 1 中，布局在编辑器中看起来很完美，但视图 C 上却没有垂直约束条件。在设备上绘制此布局时，虽然视图 C 与视图 A 的左右边缘水平对齐，但由于没有垂直约束条件，它会显示在屏幕顶部。

![img](https://developer.android.com/training/constraint-layout/images/constraint-fail_2x.png?hl=zh_cn)

* **图 1.** 编辑器将视图 C 显示在视图 A 下方，但它并没有垂直约束条件

![img](https://developer.android.com/training/constraint-layout/images/constraint-fail-fixed_2x.png?hl=zh_cn)

* **图 2.** 视图 C 现垂直约束在视图 A 下方

* 尽管缺少约束条件不会导致出现编译错误，但布局编辑器会将缺少约束条件作为错误显示在工具栏中。要查看错误和其他警告，请点击 **Show Warnings and Errors**![img](https://developer.android.com/studio/images/buttons/layout-editor-errors.png?hl=zh_cn)。 为帮助您避免出现缺少约束条件这一问题，布局编辑器会使用 [Autoconnect 和 Infer Constraints](https://developer.android.com/training/constraint-layout?hl=zh_cn#use-autoconnect-and-infer-constraints) 功能自动为您添加约束条件。

## 二、将 ConstraintLayout 添加到项目中

* 如需在项目中使用 `ConstraintLayout`，请按以下步骤操作：

1. 确保您的`maven.google.com`代码库已在模块级`build.gradle`文件中声明：

   ```groovy
       repositories {
           google()
       }
       
   ```

2. 将该库作为依赖项添加到`build.gradle`文件中，如以下示例所示。请注意，最新版本可能与示例中显示的不同：

   ```groovy
   dependencies {
       implementation "androidx.constraintlayout:constraintlayout:2.1.0"
       // To use constraintlayout in compose
       implementation "androidx.constraintlayout:constraintlayout-compose:1.0.0-beta02"
   }
   ```

3. 在工具栏或同步通知中，点击 **Sync Project with Gradle Files**。

* 现在，您可以使用 `ConstraintLayout` 构建布局。

### 2.1 转换布局

<img src="https://developer.android.com/training/constraint-layout/images/layout-editor-convert-to-constraint_2x.png?hl=zh_cn" alt="img" style="zoom:67%;" />

* **图 3.** 用于将布局转换为 `ConstraintLayout` 的菜单

* 如需将现有布局转换为约束布局，请按以下步骤操作：

1. 在 Android Studio 中打开您的布局，然后点击编辑器窗口底部的 **Design** 标签页。
2. 在 **Component Tree** 窗口中，右键点击该布局，然后点击 **Convert \*layout\* to ConstraintLayout**。

### 2.2 创建新布局

* 如需开始新的约束布局文件，请按以下步骤操作：

1. 在 **Project** 窗口中，点击模块文件夹，然后依次选择 **File > New > XML > Layout XML**。
2. 输入该布局文件的名称，并对 **Root Tag** 输入“androidx.constraintlayout.widget.ConstraintLayout”。
3. 点击 **Finish**。

## 三、添加或移除约束条件

* 如需添加约束条件，请执行以下操作：

<video src="https://developer.android.com/images/training/constraint-layout/constraint-layout-constrain-left.mov?hl=zh_cn" controls="" style="box-sizing: inherit; border: 0px; height: auto; max-width: 100%;"></video>

* **视频 1.** 视图的左侧被约束在父布局的左侧

1. 将视图从 **Palette** 窗口拖到编辑器中。

   当您在 `ConstraintLayout` 中添加视图时，该视图会显示一个边界框，每个角都有用于调整大小的方形手柄，每条边上都有圆形的约束手柄。

2. 点击视图将其选中。

3. 执行以下任一操作：

   - 点击约束手柄并将其拖动到可用定位点。 此点可以是另一视图的边缘、布局的边缘或者引导线。请注意，当您拖动约束手柄时，布局编辑器会显示可行的连接定位点和蓝色叠加层。

   - 点击 **Attributes** 窗口的 **Layout** 部分中的 **Create a connection** 按钮 ![img](https://developer.android.com/studio/images/buttons/attributes-plus-icon_2x.png?hl=zh_cn)，具体如图 4 所示。

     <img src="https://developer.android.com/images/training/constraint-layout/constraint-layout-create-constraint-2x.png?hl=zh_cn" alt="img" style="zoom:67%;" />

* 创建约束条件后，编辑器会为其指定[默认外边距](https://developer.android.com/training/constraint-layout?hl=zh_cn#adjust-the-view-margins)来分隔两个视图。

* 创建约束条件时，请注意以下规则：
  * 每个视图都必须至少有两个约束条件：一个水平约束条件，一个垂直约束条件。
  * 您只能在共用同一平面的约束手柄与定位点之间创建约束条件。因此，视图的垂直平面（左侧和右侧）只能约束在另一个垂直平面上；而基准线则只能约束到其他基准线上。
  * 每个约束句柄只能用于一个约束条件，但您可以在同一定位点上创建多个约束条件（从不同的视图）。

* 您可以通过执行以下任一操作来删除约束条件：
  * 点击某个约束条件将其选中，然后按 `Delete`。
  * 按住 `Control`（在 macOS 上为 `Command`），然后点击某个约束定位点。请注意，该约束条件变为红色即表示您可以点击将其删除，如图 5 所示。

  <img src="https://developer.android.com/images/training/constraint-layout/constraint-layout-delete-1-2x.png?hl=zh_cn" alt="img" style="zoom:67%;" />

* 在 **Attributes** 窗口的 **Layout** 部分中，点击某个约束定位点，如图 6 所示。

  <img src="https://developer.android.com/images/training/constraint-layout/constraint-layout-delete-2-2x.png?hl=zh_cn" alt="img" style="zoom:67%;" />

* 如果您在视图中添加了相反的约束条件，则约束线会像弹簧一样弯弯曲曲，以此来表示相反的力，如视频 2 中所示。当视图尺寸设置为“fixed”或“wrap content”时，效果最明显，在这种情况下，视图在两个约束条件之间居中。如果您希望视图扩展其尺寸以满足约束条件，则可以[将尺寸切换为“match constraints”](https://developer.android.com/training/constraint-layout?hl=zh_cn#adjust-the-view-size)；如果您希望保持当前尺寸，但想要移动视图使其不要居中，则可以[调整约束偏差](https://developer.android.com/training/constraint-layout?hl=zh_cn#adjust-the-constraint-bias)。

<video src="https://developer.android.com/images/training/constraint-layout/constraint-layout-constrain-left.mov?hl=zh_cn" controls="" style="box-sizing: inherit; border: 0px; height: auto; max-width: 100%;"></video>

* **视频 2.** 添加与现有约束条件相反的约束条件

* 您可以使用约束条件来实现不同类型的布局行为，如以下部分所述。

### 3.1 父级位置

* 将视图的一侧约束到布局的相应边缘。

* 在图 7 中，视图的左侧连接到父布局的左边缘。您可以使用外边距来定义距离边缘的距离。



![img](https://developer.android.com/training/constraint-layout/images/parent-constraint_2x.png?hl=zh_cn)

* **图 7.** 对父级的水平约束

### 3.2 顺序位置

* 定义两个视图的显示顺序（垂直或水平方向）。

* 在图 8 中，B 被约束为始终位于 A 的右侧，而 C 被约束在 A 的下方。 不过，这些约束条件并不意味着对齐，因此 B 仍然可以上下移动。

![img](https://developer.android.com/training/constraint-layout/images/position-constraint_2x.png?hl=zh_cn)

* **图 8.** 水平和垂直约束方式

### 3.3 对齐方式

* 将一个视图的边缘与另一视图的同一边对齐。

* 在图 9 中，B 的左侧与 A 的左侧对齐。 如果要与视图中心对齐，请对两侧创建约束条件。

* 您可以通过从约束布局向内拖动视图来偏移对齐量。例如，图 10 显示 B 的偏移对齐为 24dp。 偏移量由受约束视图的外边距定义。

* 您还可以选择要对齐的所有视图，然后点击工具栏中的 **Align** 图标 ![img](https://developer.android.com/studio/images/buttons/layout-editor-align.png?hl=zh_cn) 以选择对齐类型。

![img](https://developer.android.com/training/constraint-layout/images/alignment-constraint_2x.png?hl=zh_cn)

* **图 9.** 水平对齐约束

![img](https://developer.android.com/training/constraint-layout/images/alignment-constraint-offset_2x.png?hl=zh_cn)

* **图 10.** 偏移水平对齐约束

### 3.4 基线对齐

* 将一个视图的文本基线与另一视图的文本基线对齐。

* 在图 11 中，B 的第一行与 A 中的文本对齐。

* 要创建基线约束条件，请右键点击要约束的文本视图，然后点击 **Show Baseline**。接着点击文本基线并将其拖到另一基线上。

![img](https://developer.android.com/training/constraint-layout/images/baseline-constraint_2x.png?hl=zh_cn)

* **图 11.** 基线对齐约束

### 3.5 引导线约束

* 您可以添加垂直或水平的引导线来约束视图，并且应用用户看不到该引导线。 您可以根据相对于布局边缘的 dp 单位或百分比在布局中定位引导线。

* 要创建引导线，请点击工具栏中的 **Guidelines**![img](https://developer.android.com/studio/images/buttons/layout-editor-guidelines.png?hl=zh_cn)，然后点击 **Add Vertical Guideline** 或 **Add Horizontal Guideline**。

* 拖动虚线将其重新定位，然后点击引导线边缘的圆圈以切换测量模式。

![img](https://developer.android.com/training/constraint-layout/images/guideline-constraint_2x.png?hl=zh_cn)

* **图 12.** 受引导线约束的视图

### 3.6 屏障约束

* 与引导线类似，屏障是一条隐藏的线，您可以用它来约束视图。屏障不会定义自己的位置；相反，屏障的位置会随着其中所含视图的位置而移动。如果您希望将视图限制到一组视图而不是某个特定视图，这就非常有用。

* 例如，图 13 显示视图 C 被约束在屏障的右侧。该屏障设置为视图 A 和视图 B 的“end”侧（或从左至右布局中的右侧）。因此，屏障根据视图 A 或视图 B 的右侧是否为最右侧来移动。

* 如需创建屏障，请按以下步骤操作：

1. 点击工具栏中的 **Guidelines** 图标 ![img](https://developer.android.com/studio/images/buttons/layout-editor-guidelines.png?hl=zh_cn)，然后点击 **Add Vertical Barrier** 或 **Add Horizontal Barrier**。
2. 在 **Component Tree** 窗口中，选择要放入屏障内的视图，然后将其拖动到屏障组件中。
3. 在 **Component Tree** 中选择障碍，打开 **Attributes**![img](https://developer.android.com/studio/images/buttons/window-properties.png?hl=zh_cn) 窗口，然后设置 **barrierDirection**。

* 现在，您可以从另一个视图创建屏障约束。

* 您还可以将屏障*内*的视图约束到屏障。这样，您就可以确保屏障中的所有视图始终相互对齐，即使您并不知道哪个视图最长或最高。

* 您还可以在屏障内添加引导线，以确保屏障的位置“最小”。

<img src="https://developer.android.com/training/constraint-layout/images/barrier-constraint_2x.png?hl=zh_cn" alt="img" style="zoom:67%;" />

* **图 13.** 视图 C 约束在屏障内，该屏障会根据视图 A 和视图 B 的位置/尺寸而移动

## 四、调整约束偏差

* 对某个视图的两侧添加约束条件（并且同一维度的视图尺寸为“fixed”或者“wrap Content”）时，则该视图在两个约束条件之间居中且默认偏差为 50%。您可以通过拖动 **Attributes** 窗口的偏差滑块或拖动视图来调整偏差，如视频 3 中所示。

* 如果您希望视图扩展其尺寸以满足约束条件，可以[将尺寸切换为“match constraints”](https://developer.android.com/training/constraint-layout?hl=zh_cn#adjust-the-view-size)。

<video src="https://developer.android.com/images/training/constraint-layout/constraint-layout-adjust-bias.mov?hl=zh_cn" controls="" style="box-sizing: inherit; border: 0px; height: auto; max-width: 100%;"></video>

* **视频 3.** 调整约束偏差

## 五、调整视图尺寸

<img src="https://developer.android.com/images/training/constraint-layout/constraint-layout-editor-attributes-2x.png?hl=zh_cn" alt="img" style="zoom:67%;" />

* **图 14.** 在选择视图时，**Attributes** 窗口会包含如下控件：1尺寸比、2 删除约束条件、3 高度/宽度模式、4 外边距和 5 约束偏差。您还可以通过点击 6 约束列表中的各个约束条件来突出显示布局编辑器中的各个约束条件。

* 您可以使用角手柄来调整视图的尺寸，但这会对尺寸进行硬编码，从而使视图不会针对不同的内容或屏幕尺寸进行调整。要选择不同的尺寸模式，请点击视图，然后打开编辑器右侧的 **Attributes**![img](https://developer.android.com/studio/images/buttons/window-properties.png?hl=zh_cn) 窗口。

* **Attributes** 窗口顶部附近的视图检查器中包括若干布局属性的控件，如图 14 所示（仅适用于约束布局中的视图）。

* 您可以通过点击图 14 中标注 3 所指示的符号来更改高度和宽度的计算方式。这些符号代表如下所示的尺寸模式（点击符号即可切换这些设置）：
  * ![img](https://developer.android.com/studio/images/buttons/layout-width-fixed.png?hl=zh_cn) **Fixed**：您可以在下面的文本框中指定具体维度，也可以在编辑器中调整视图尺寸。

  * ![img](https://developer.android.com/studio/images/buttons/layout-width-wrap.png?hl=zh_cn)**Wrap Content**：视图仅在需要时扩展以适应其内容。

  * ![img](https://developer.android.com/studio/images/buttons/layout-width-match.png?hl=zh_cn) **Match Constraints**：视图会尽可能扩展，以满足每侧的约束条件（在考虑视图的外边距之后）。不过，您可以使用以下属性和值修改该行为（这些属性仅在您将视图宽度设置为“match constraints”时才会生效)：

    * layout_constraintWidth_default

      - **spread**：尽可能扩展视图以满足每侧的约束条件。这是默认行为。
      - **wrap**：仅在需要时扩展视图以适应其内容，但如有约束条件限制，视图仍然可以小于其内容。因此，它与使用 **Wrap Content**（上面）之间的区别在于，将宽度设为 **Wrap Content** 会强行使宽度始终与内容宽度完全匹配；而使用 **layout_constraintWidth_default** 设置为 **wrap** 的 **Match Constraints** 时，视图可以小于内容宽度。

    * layout_constraintWidth_min

      该视图的最小宽度采用 `dp` 维度。

    * layout_constraintWidth_max

      该视图的最大宽度采用 `dp` 维度。

    不过，如果给定维度只有一个约束条件，则视图会扩展以适应其内容。 在高度或宽度上使用此模式也可让您[设置尺寸比](https://developer.android.com/training/constraint-layout?hl=zh_cn#ratio)。

* **注意**：您无法将 `match_parent` 用于 `ConstraintLayout` 中的任何视图。请改用“match constraints” (`0dp`)。

### 5.1 将尺寸设置为比率

* 如果至少有一个视图尺寸设置为“match constraints”(`0dp`)，您可以将视图尺寸设置为 16:9。如需启用该比率，请点击 **Toggle Aspect Ratio Constraint**（图 14 中的标注 1），然后在出现的输入框中输入 width:height 比率。

* 如果宽度和高度都设置为“match constraints”，您可以点击 **Toggle Aspect Ratio Constraint**，选择哪个维度基于与另一个维度的比率。 视图检查器通过用实线连接相应的边缘来指明哪个被设为比率。

* 例如，如果您将两侧都设置为“match constraints”，请双击 **Toggle Aspect Ratio Constraint**，将宽度设置为与高度的比率。现在整个尺寸由视图的高度决定（可以以任意方式定义），如图 15 所示。

<img src="https://developer.android.com/images/training/constraint-layout/constraint-layout-ratio-2x.png?hl=zh_cn" alt="img" style="zoom:67%;" />

* **图 15.** 该视图的宽高比设置为 16：9，其宽度基于与高度的比率。

## 六、调整视图外边距

* 要确保所有视图间隔均匀，请点击工具栏中的 **Margin**![img](https://developer.android.com/images/training/constraint-layout/constraint-layout-margin-icon-2x.png?hl=zh_cn)，为您添加到布局的每个视图选择默认外边距。 您对默认外边距所做的任何更改仅应用于您在更改后添加的视图。

* 您可以通过点击代表每个约束条件所在行的数字来控制 **Attributes** 窗口中每个视图的外边距（在图 14 中，标注 4 表明下外边距设置为 16dp）。

![img](https://developer.android.com/images/training/constraint-layout/constraint-layout-margin-2x.png?hl=zh_cn)

* **图 16.** 工具栏的 **Margin** 按钮。

* 工具提供的所有外边距均为 8dp 的倍数，以确保您的视图与 Material Design 提供的 [8dp 方形网格建议](https://material.google.com/layout/metrics-keylines?hl=zh_cn)保持一致。

## 七、使用链控制线性组

* 链是一组视图，这些视图通过双向位置约束条件相互链接到一起。链中的视图可以垂直或水平分布。

* 链可以采用以下几种样式之一：

1. **Spread**：视图是均匀分布的（在考虑外边距之后）。这是默认值。

![img](https://developer.android.com/training/constraint-layout/images/constraint-chain_2x.png?hl=zh_cn)

1. **Spread inside**：第一个和最后一个视图固定在链两端的约束边界上，其余视图均匀分布。
2. **Weighted**：当链设置为 **spread** 或 **spread inside** 时，您可以通过将一个或多个视图设置为“match constraints”(`0dp`) 来填充剩余空间。默认情况下，设置为“match constraints”的每个视图之间的空间均匀分布，但您可以使用 `layout_constraintHorizontal_weight` 和 `layout_constraintVertical_weight` 属性为每个视图分配重要性权重。如果您熟悉[线性布局](https://developer.android.com/guide/topics/ui/layout/linear?hl=zh_cn)中的 `layout_weight` 的话，就会知道该样式与它的原理是相同的。因此，权重值最高的视图获得的空间最大；相同权重的视图获得同样大小的空间。
3. **Packed**：视图打包在一起（在考虑外边距之后）。 然后，您可以通过更改链的头视图偏差调整整条链的偏差（左/右或上/下）。

![img](https://developer.android.com/training/constraint-layout/images/constraint-chain-styles_2x.png?hl=zh_cn)

* **图 18.** 每种链样式的示例

* 链的“头”视图（水平链中最左侧的视图以及垂直链中最顶部的视图）以 XML 格式定义链的样式。不过，您可以通过选择链中的任意视图，然后点击出现在该视图下方的链按钮 ![img](https://developer.android.com/studio/images/buttons/layout-editor-action-chain.png?hl=zh_cn)，在 **spread**、**spread inside** 和 **packed** 之间进行切换。

* 如需创建链，请选择要包含在链中的所有视图，右键点击其中一个视图，选择 **Chains**，然后选择 **Center Horizontally** 或 **Center Vertically**，如视频 4 中所示：

<video src="https://developer.android.com/images/training/constraint-layout/constraint-layout-chain.mov?hl=zh_cn" controls="" style="box-sizing: inherit; border: 0px; height: auto; max-width: 100%;"></video>

* **视频 4.** 创建水平链

* 以下是使用链时需要考虑的其他事项：
  * 视图可以是水平链和垂直链的一部分，因此可以轻松构建灵活的网格布局。
  * 只有当链的每一端都被约束到同一轴上的另一个对象时，链才能正常工作，如图 14 所示。
  * 虽然链的方向为垂直或水平，但使用其中一个方向不会沿该方向与视图对齐。因此，请务必包含其他约束条件，以便使链中的每个视图都能正确定位，例如[对齐约束](https://developer.android.com/training/constraint-layout?hl=zh_cn#alignment)。

## 八、自动创建约束条件

* 您可以将每个视图移动到您希望的位置，然后点击 **Infer Constraints**![img](https://developer.android.com/studio/images/buttons/layout-editor-infer.png?hl=zh_cn) 自动创建约束条件，而不是在将视图放入布局中时，为其添加约束条件。

* **Infer Constraints** 会扫描布局，以便为所有视图确定最有效的约束集。它会尽可能将视图约束在当前位置，同时提高灵活性。您可能需要进行一些调整，以确保布局能够按照您的预期针对不同的屏幕尺寸和方向进行响应。

* **Autoconnect to parent** 是您可以启用的独立功能。启用后，当您将子视图添加到父视图时，此功能会自动为每个视图创建两个或多个约束条件，但仅在可以将视图约束到父布局的情况。Autoconnect 不会为布局中的其他视图创建约束条件。

* Autoconnect 在默认情况下处于停用状态。点击布局编辑器工具栏中的 **Enable Autoconnection to Parent**![img](https://developer.android.com/studio/images/buttons/layout-editor-autoconnect-on.png?hl=zh_cn)，即可启用该功能。

## 九、关键帧动画

* 在 `ConstraintLayout` 中，您可以使用 [ConstraintSet](https://developer.android.com/reference/androidx/constraintlayout/widget/ConstraintSet?hl=zh_cn) 和 [TransitionManager](https://developer.android.com/reference/android/transition/TransitionManager?hl=zh_cn) 为尺寸和位置元素的变化添加动画效果。

* **注意**：`TransitionManager` 在 Android 4.0 （API 级别 14）或更高级别的支持库中提供。

* `ConstraintSet` 是一个轻量型对象，表示 `ConstraintLayout` 内所有子元素的约束条件、外边距和内边距。当您将 `ConstraintSet` 应用于显示的 `ConstraintLayout` 时，布局会更新其所有子级的约束条件。

* 要使用 ConstraintSets 制作动画，请指定两个布局文件作为动画的开始和结束关键帧。然后，您可以从第二个关键帧文件加载 `ConstraintSet` 并将其应用于显示的 `ConstraintLayout` 中。

* **重要提示**：ConstraintSet 动画仅为子元素的尺寸和位置添加动画效果。它们不会为其他属性（例如颜色）添加动画效果。

* 下面的代码示例显示了如何为一个按钮移动到屏幕底部添加动画效果。

```kotlin
// MainActivity.kt

    fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.keyframe_one)
        constraintLayout = findViewById(R.id.constraint_layout) // member variable
    }

    fun animateToKeyframeTwo() {
        val constraintSet = ConstraintSet()
        constraintSet.load(this, R.layout.keyframe_two)
        TransitionManager.beginDelayedTransition()
        constraintSet.applyTo(constraintLayout)
    }
    
    // layout/keyframe1.xml
    // Keyframe 1 contains the starting position for all elements in the animation as well as final colors and text sizes

    <?xml version="1.0" encoding="utf-8"?>
    <androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <Button
            android:id="@+id/button2"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:text="Button"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />
    </androidx.constraintlayout.widget.ConstraintLayout>
    
    // layout/keyframe2.xml
    // Keyframe 2 contains another ConstraintLayout with the final positions

    <?xml version="1.0" encoding="utf-8"?>
    <androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:app="http://schemas.android.com/apk/res-auto"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <Button
            android:id="@+id/button2"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:text="Button"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintBottom_toBottomOf="parent" />
    </androidx.constraintlayout.widget.ConstraintLayout>

```

* **注意**：在 Android Studio 3.6 及更高版本中，[视图绑定](https://developer.android.com/topic/libraries/view-binding?hl=zh_cn)功能可以替换 `findViewById()` 调用，并为与视图互动的代码提供编译时类型安全。考虑使用视图绑定，而非 `findViewById()`。

## 十、其他资源

* `ConstraintLayout` 用于 [Sunflower](https://github.com/android/sunflower) 演示版应用。