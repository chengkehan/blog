# 矩阵相乘优化

**2017-4-21**

定义两个矩阵：

	uniform float4x4 mat0;	uniform float4x4 mat1;
	
利用这两个的累积变换，对一个点进行变换操作，可以是类似下面这样：

	o.vertex = mul(mat0, mul(mat1, v.vertex));
	
也可以是这样：

	o.vertex = mul(mul(mat0, mat1), v.vertex);
	
这两种方式得到的结果是一样的，但是计算量却相差很大。最简单的验证方式就是手工计算一遍，就能体会到了。另一种更直观的方式，可以看一下对应的 glsl。

第一种：
	
	// cg
	o.vertex = mul(mat0, mul(mat1, v.vertex));
	// glsl
	u_xlat0 = in_POSITION0.yyyy * hlslcc_mtx4x4mat1[1];
	u_xlat0 = hlslcc_mtx4x4mat1[0] * in_POSITION0.xxxx + u_xlat0;
	u_xlat0 = hlslcc_mtx4x4mat1[2] * in_POSITION0.zzzz + u_xlat0;
	u_xlat0 = hlslcc_mtx4x4mat1[3] * in_POSITION0.wwww + u_xlat0;
	u_xlat1 = u_xlat0.yyyy * hlslcc_mtx4x4mat0[1];
	u_xlat1 = hlslcc_mtx4x4mat0[0] * u_xlat0.xxxx + u_xlat1;
	u_xlat1 = hlslcc_mtx4x4mat0[2] * u_xlat0.zzzz + u_xlat1;
	gl_Position = hlslcc_mtx4x4mat0[3] * u_xlat0.wwww + u_xlat1;
	
第二种：

	// cg
	o.vertex = mul(mul(mat0, mat1), v.vertex);
	// glsl
    u_xlat0 = hlslcc_mtx4x4mat0[1] * hlslcc_mtx4x4mat1[1].yyyy;
    u_xlat0 = hlslcc_mtx4x4mat0[0] * hlslcc_mtx4x4mat1[1].xxxx + u_xlat0;
    u_xlat0 = hlslcc_mtx4x4mat0[2] * hlslcc_mtx4x4mat1[1].zzzz + u_xlat0;
    u_xlat0 = hlslcc_mtx4x4mat0[3] * hlslcc_mtx4x4mat1[1].wwww + u_xlat0;
    u_xlat0 = u_xlat0 * in_POSITION0.yyyy;
    u_xlat1 = hlslcc_mtx4x4mat0[1] * hlslcc_mtx4x4mat1[0].yyyy;
    u_xlat1 = hlslcc_mtx4x4mat0[0] * hlslcc_mtx4x4mat1[0].xxxx + u_xlat1;
    u_xlat1 = hlslcc_mtx4x4mat0[2] * hlslcc_mtx4x4mat1[0].zzzz + u_xlat1;
    u_xlat1 = hlslcc_mtx4x4mat0[3] * hlslcc_mtx4x4mat1[0].wwww + u_xlat1;
    u_xlat0 = u_xlat1 * in_POSITION0.xxxx + u_xlat0;
    u_xlat1 = hlslcc_mtx4x4mat0[1] * hlslcc_mtx4x4mat1[2].yyyy;
    u_xlat1 = hlslcc_mtx4x4mat0[0] * hlslcc_mtx4x4mat1[2].xxxx + u_xlat1;
    u_xlat1 = hlslcc_mtx4x4mat0[2] * hlslcc_mtx4x4mat1[2].zzzz + u_xlat1;
    u_xlat1 = hlslcc_mtx4x4mat0[3] * hlslcc_mtx4x4mat1[2].wwww + u_xlat1;
    u_xlat0 = u_xlat1 * in_POSITION0.zzzz + u_xlat0;
    u_xlat1 = hlslcc_mtx4x4mat0[1] * hlslcc_mtx4x4mat1[3].yyyy;
    u_xlat1 = hlslcc_mtx4x4mat0[0] * hlslcc_mtx4x4mat1[3].xxxx + u_xlat1;
    u_xlat1 = hlslcc_mtx4x4mat0[2] * hlslcc_mtx4x4mat1[3].zzzz + u_xlat1;
    u_xlat1 = hlslcc_mtx4x4mat0[3] * hlslcc_mtx4x4mat1[3].wwww + u_xlat1;
    gl_Position = u_xlat1 * in_POSITION0.wwww + u_xlat0;
    
这样很直观的可以看到指令数量上的差别，指令越多也就意味着计算量越大，GPU 的压力也越大，但是两种方法得到的结果是一样的，所以这一点在优化时需要注意了。

Unity 提供的 Shader 工具库也有一处同样的优化，以前在将顶点从模型空间变换到齐次空间时是这么写的：

	o.vertex = mul(UNITY_MATRIX_MVP, v.vertex);
	
其实现在也可以这么写，一点问题也没有。但是 Unity 为我们提供了一个新的方法：

	inline float4 UnityObjectToClipPos( in float3 pos )
	{
	#ifdef UNITY_USE_PREMULTIPLIED_MATRICES
		return mul(UNITY_MATRIX_MVP, float4(pos, 1.0));
	#else
		return mul(UNITY_MATRIX_VP, mul(unity_ObjectToWorld, float4(pos, 1.0)));
	#endif
	}
	inline float4 UnityObjectToClipPos(float4 pos)
	{
		return UnityObjectToClipPos(pos.xyz);
	}

利用这个新的方法，现在推荐的写法是：

	o.vertex = UnityObjectToClipPos(v.vertex);
	
为什么推荐这么写呢？其实如果没有用到 GPU Instancing 时，新写法和老写法没有什么差别，但是当开启 GPU Instancing 时，使用新的写法就能减少相当的计算量，具体能减少多上，从上面的 glsl 对比就能很直观的看出来了，当一帧渲染的顶点很多时这个开销就非常可观了。那这为什么和 GPU Instancing 有关呢？其实 Unity 在背后为我们做了很多工作。

当开启 GPU Instancing 时，`UNITY_MATRIX_MVP` 会被解释为 `mul(UNITY_MATRIX_VP, unity_ObjectToWorld)`，而 `unity_ObjectToWorld` 又会被解释为 `unity_ObjectToWorldArray[unity_InstanceID]`。这样 `UNITY_MATRIX_MVP` 不再是一个 uniform 常量，而变成了两个矩阵相乘，所以就需要像最开始的代码那样进行优化，这就是关于 UnityObjectToClipPos 的故事。
	

	