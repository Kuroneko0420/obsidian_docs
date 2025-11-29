1。返回值类型
	transformation类型的算子是返回一个新的rdd
	action算子返回的是一个非分布式的结果，通常是scala集合类型，或者基本数据类型
2.惰性执行
	transformation类型的算子通常是惰性执行，遇到action类型的算子才会执行触发
3.操作触发时机
	transformation类型的算子通常在数据集上定义了一系列的转换操作，而不会立即触发计算。只有当遇到action类型的算子时，spark才会根据····