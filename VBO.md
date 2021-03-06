#Vertex Buffer Object

**2016-3-2**

Vertex Buffer Object 简称为 VBO，它的作用是在显存中开辟一块区域，用于存放数据，这块数据里面存储了模型顶点相关的数据，所以我们把它叫做顶点缓冲对象。如果存储的是其他类型的数据，那就叫其它名字了。其实对于 GPU 来说，显存中存储了什么类型的数据并不是最重要的，重要的是 GPU 拿到了这个数据之后要怎么解析这些数据。因此在更现代的图形 API 中，向显存中存储数据时将不再纠结于具体的类型，而是在使用读取数据的时候使用不同的解析器来解析。这将大大提高底层图形接口的效率。

为什么会有 VBO 呢？以前需要渲染一个模型的时候，总是直接把所有顶点数据直接从内存传送到显存中，每一帧都是如此。这样就对系统此层造成了很大的压力。于是就考虑是否可以将这些需要传送的数据直接的存放在显存中，当需要渲染的时候就不需要再传送一遍了，GPU 直接取数据就行了，这样就有了 VBO。

VBO 是底层图形 API 的称呼，那么对于我们开发者来说，VBO 一般会被引擎进行抽象的表达，这里我们把它叫做 Mesh。Mesh 里面包含了 vertices,indices,normal,tangents,uv等，有了这些信息就可以完整的表达一个模型了。有的时候一个 Mesh 并不一定只是一个 VBO，可能是由好几个 VBO 组成。当一个 Mesh 里只有一个 VBO 时，那么这个 VBO 就包含了描述这个模型的所有数据。当一个 Mesh 里有多个 VBO 时，引擎的设计者会根据具体的需要，将不同需求的数据存放到各自的 VBO 中。这样做有什么好处呢，下面举例说明。

有些情况下，3D 模型并不是完全静态的，顶点数据需要在运行时进行更新，如果这种数据的变化可以用某种公式表达出来，那就可以尝试着在 Shader 中计算新的数据，而不是去刷新 VBO，但是如果无法在 Shader 中计算出新的数据，那么就只能刷新 VBO 了。这时，如果所有的数据都放在了一个 VBO 中，就必须刷新整个 VBO（先修改内存中的数据，然后再将数据提交到显存），如果将需要刷新的数据单独放到一个 VBO 中，就只需要刷新一小块数据，底层的效率就会高很多。在一些高级的图形接口中可以映射一个 VBO 的一部分到内存中，并且部分设备是内存显存共享，还可以避免内存拷贝的消耗，只要使用得当也可以规避单 VBO 的缺陷。

有一种情况使得注意，可能会成为一个潜在的问题。就是当前帧 GPU 整在读取 VBO 数据时，CPU 这时正在进行下一帧的计算，而计算步骤中需要修改VBO，但是这时 GPU 正在读取这个 VBO，这时底层接口就有可能会暂时挂起 CPU ，直到 GPU 完整当前帧。