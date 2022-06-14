+ Basic mathematics
	+ Linear algebra, calculus, statistics

+ Basic physics
	+ Optics, Mechanics

+ Misc
	+ Signal processing
	+ Numerical analysis

*And a bit of aesthetics*

+ More dependent on Linear Algebra
	+ Vectors(dot products, cross products, ...)
	+ Matrices(matrix-matrix, matrix-vector mult., ...)

## Vector
### Dot Product
$\vec{a} \cdot \vec{b} = \mid\mid \vec{a} \mid\mid\mid\mid \vec{b}\mid\mid\cos\Theta$

图形学中主要应用
得出向量之间的夹角，判断两个向量是否接近
视觉向量和光向量的夹角来判断反光效果

### Cross Product
$\vec{a} \times \vec{b} = - \vec{b} \times \vec{a}$
$\mid \mid \vec{a} \times \vec{b}\mid\mid = \mid\mid \vec{a}\mid\mid\mid\mid\vec{b}\mid\mid\sin\Theta$
![[Pasted image 20220613152422.png]]
+ Cross product is orthogonal to two initial vectors
+ Direction determined by right-hand rule
+ Useful in constructing coordinate systems(later)

*Cross product: Properties*
![[Pasted image 20220613152651.png]]

*Cross Product: Cartesian Formula*
![[Pasted image 20220613153417.png]]

*Cross PRoduct in Graphics*
+ Determine left / right
+ Determine inside / outside

如果 $\vec{a} \times \vec{b}$是正的，说明$\vec{b}$在$\vec{a}$的左侧

*怎么判断p点在三角形内部？*
![[Pasted image 20220613154143.png]]
p点分别与三个顶点连线，然后与三条边进行叉乘，如果同时都在边向量的右侧（或者左侧）说明点p在三角形的内部

*Orthonormal Coordinate Frames*
![[Pasted image 20220613154441.png]]

## Matrices
+ magical 2D arrays that haunt in every CS course
+ In Graphics, pervasively used to represent transformations
	+ Translation, rotation, shear, scale


### Matrix-Matrix Mltiplication
+ #(number of ) columns in A must = # rows in B
	(M x N)(Nx P) = (M x P)
+ Element(i,j) in the product is the dot product of row i from A and column j from B  
+ Properties
	+ Non-commutative 
		(AB and BA are different in general)
	+ Associative and distributive
		+ (AB)C = A(BC)
		+ A(B+C) = AB + AC
		+ (A+B)C = AC + BC 

### Matrix-Vector Multiplication
+ Treat vector as a column matrix(m * 1)

### Transpose of a Matrix
![[Pasted image 20220613160211.png]]

### Identity Matrix and Inverses
![[Pasted image 20220613160326.png]]

### Vector multiplication in Matrix form
![[Pasted image 20220613160518.png]]


##
### Shear Matrix
### Rotate(about the origin (0,0), CCW by default)
### Linear Transforms = Matrices
![[Pasted image 20220613165401.png]]
### Solution: Homogenous Coordinates
![[Pasted image 20220613174342.png]]
![[Pasted image 20220613174610.png]]
### Inverse Transform
![[Pasted image 20220613175245.png]]
### Composite Transform

## Viewing transformation
+ View / Camera transformation
+ Projection transformation
	+ Orthographic projection
	+ Perspective projection

## 3D Transformations
### Rotation around x, y, or z-axis
![[Pasted image 20220613203318.png]]

+ Compose any 3D rotation from Rx, Ry, Rz
	+ ![[Pasted image 20220613203819.png]]
	+ So-calles Euler angles
	+ Often used in flight simulators: roll, pitch, yaw

### Rodrigues' Rotation Formula
![[Pasted image 20220613204119.png]]
### View / Camera Transformation
+ What is view transformation?
+ Think about how to take photo
	+ Find a good place and arrange people(model transformation)
	+ Find a good "angle" to put the camera(view transformation)
	+ Cheese!(projection transformation)

+ Hw to perform view transformation?
+ Define the camera first
	+ Position   $\vec{e}$
	+ Look-at / gaze direction  $\hat{g}$
	+ up direction $\hat{t}$
		(assuming perp. to loot-at)
+ Key observation
	+ if the camera and all objects move together, the "photo" will be the same

+ Transform the camera by Mview
	+ so it's located at the origin, up at Y, look at -Z

+ Mview in math
	+ Translates e to origin
	+ Rotates g to -Z
	+ Rotates t toY
	+ Rotates(G $\times$ t)To X

+ Mview in math
![[Pasted image 20220613211427.png]]

+ Summary
	+ Transform objects together with the camera
	+ Until camera's an the origin, up at Y, look at -Z

+ Also known as ModelView Transformation

### Projection Transformation
+ Projection in Computer Graphics
	+ 3D to 2D
	+ Orthographic projection
	+ Perspective projection

+ Perspective projection vs. Orthographic projection

+ Orthographic Projection
	+ A simple way of understanding
		+ Camera located at origin, looking an -Z, up at Y(looks familiar)
		+ Drop Z coordinate
		+ Translate and scale the resulting rectangle to [-1, 1]

+ In general
	+ We want to map a cuboid [l, r] $\times$[b, t]$\times$[f.n]to the "canonical" cube
	(l for left, r for rigth, b for below, t for top, f for far, n for near)
	+ Slightly different orders(to the "Simple way")
		+ Center cuboid by translating
		+ Scale into "canonical" cube

+ Transformation matrix
	+ ![[Pasted image 20220613213146.png]]

## Perspective Projection
+ Most common in Computer Graphics, art, visual system
+ Further objects are smaller
+ Parallel lines not parallel; converge to single point
+ ![[Pasted image 20220613213613.png]]

+ How to do perspective projection
	+ First "squish" the frustum into a cuboid(n->n, f->f)
	+ Do orthographic projection
	+ ![[Pasted image 20220613214449.png]]
+ In order to find a transformation
	+ Recall the key idea: findt the relationship between transformed points(x', y', z') and the original points (x, y, z)
	+ ![[Pasted image 20220613214850.png]]
	+	![[Pasted image 20220613214912.png]]
+ So the "squish" projection does this
	+ ![[Pasted image 20220613215113.png]]
+ Already good enough to figure out part of M(persp->ortho)
	+ ![[Pasted image 20220613215202.png]]
+ Any point on the near plane will not change
	+ ![[Pasted image 20220613220548.png]]
+ So the third row must be of the form (0 0 A B)
	+ ![[Pasted image 20220613220613.png]]
+ what we have now
	+ ![[Pasted image 20220613220706.png]]
+ Any point's z on the far plane will not change
	+ ![[Pasted image 20220613220736.png]]
+ Solve for A and B
	+ ![[Pasted image 20220613221019.png]]


### Frustum->Cuboid时Z如何变化

$M_{persp->ortho}$ ,将nf中间平面的中点带入转换公式：
$$
M_{persp->ortho}
\left(
\begin{matrix}
0\\
0\\
\frac{n+f}{2}\\
1
\end{matrix}
\right)
=
\left(
\begin{matrix}
0\\
0\\
\frac{n^2+f^2}{2}\\
\frac{f+n}{2}
\end{matrix}
\right)
\Rightarrow
\left(
\begin{matrix}
0\\
0\\
\frac{n^2+f^2}{f+n}\\
1
\end{matrix}
\right)
$$
即判断$\frac{n^2+f^2}{f+n}$与$\frac{n+f}{2}$的数值大小
因为方向是Z轴的负方向，n与f都是负数，且两者不相等，所以
$$
\frac{n^2+f^2}{f+n} - \frac{n+f}{2} < 0
$$
恒成立，所以Z点相比于之前的位置变远了。 
