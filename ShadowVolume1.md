# Shadow Volume 1

**2017-9-4**

Shadow Volume 是比较古老的阴影技术了，在使用的时候有很多的限制和注意点。虽然和现代阴影技术相比有这各种缺点，但在特定的条件下合理使用也不失为一种解决方案。

目前有如下的应用场景。在移动设备上大型复杂场景中的阴影，一般都是烘焙到 Lightmap。这时 Lightmap 的分辨率就直接决定了烘焙的阴影效果，如果 Lightmap 偏小，阴影边缘会非常的模糊，这会导致整个画面显得很脏。而过大的 Lightmap 分辨无疑会增加相当多的渲染开销。在 Unity 中可以使用参数调整模型占 Lightmap 的比重，以此来提高重要物件的光照效果，这种方式对于自由视角的大场景其实很难取舍，通过合理的分配后一般可以达到效果。而对于非真实渲染的时候（比如卡通渲染），画面中的阴影边缘总是希望是非常实的，而不像真实渲染的时候需要通过软阴影、半影等技术来提升效果。这时，对于稍大一点的场景，无论将 Lightmap 的分辨率增加多少，都很难达到阴影硬边的效果。

基于以上这个情况，所以我想使用 Shadow Volume 来尝试下。首先将所有的流程走通，并把可能出现的问题理清楚。这里先列出一个简单的目录，以后会基于这个目录将每点展开把思路说清楚。

* Shadow Volume 的两种实现方法及其优缺点
* 阴影边缘的软化方法
* Shadow Volume 和 Shadow Map 两种阴影融合
* 静态阴影体的生成方法

如果要集成到实际项目还需要大量的工作量，对应的工具、性能测试、效果分级等等。这里的目的主要是为了将目前的测试代码和思路进行一个有效的整理。