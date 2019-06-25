[Understanding nil - GopherCon2016](https://www.youtube.com/watch?v=ynoY2xz-F8s)
1. zero value for pointer and reference type(slice/map/channel), function, interface
2. nil is useful
	a. nil pointer as method receiver
	b. nil slice is valid zero value
	c. nil map is perfect read-only zero value
	d. essential for some concurrency patterns[close(chan) is not enough when select, need set nil]
	e. nil functions as default value and for completenesss
	f. nil interface[(type, value)] most used signal in GO(err != nil, nil != (*Person)nil)
