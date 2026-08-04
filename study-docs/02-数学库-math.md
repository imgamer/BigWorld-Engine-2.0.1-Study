# BigWorld 数学库 math 深度分析

> 源码位置：`/workspace/src/lib/math/`
> 适用版本：BigWorld Technology 2.0.1

## 1. 模块定位

### 1.1 math 是什么

`math` 是 BigWorld 引擎所有模块共享的数学库，提供 2D/3D/4D 向量、4×4 矩阵、四元数、各种包围体、平面/多边形/多面体几何、统计平均、噪声生成、颜色等基础数学类型与运算。

它的关键设计目标是：

1. **跨平台**：同一份头文件在 Win32 / Linux / Xbox360 / PlayStation3 上编译运行。
2. **基础类型可替换**：`Vector3` / `Matrix` 等的"Base"父类在不同平台可以是 D3DX 类型或自研类型，业务代码不感知。
3. **无强制 D3D 依赖**：服务端构建不链接 D3DX；客户端 Win32 构建可选地用 D3DX9 加速。
4. **SIMD 友好**：关键类型支持 16 字节对齐，提供 `VectorFast` 等显式 SIMD 变体。
5. **与 cstdmf 解耦**：仅依赖 cstdmf 的 `debug.hpp` / `stdmf.hpp` / `bwrandom.hpp` 等少量头文件。

### 1.2 模块关系图（文字描述）

```
            ┌─────────────────────────────────────────┐
            │              cstdmf                      │
            │  (debug/stdmf/bwrandom/aligned/conc...)  │
            └──────────────────┬──────────────────────┘
                               │
            ┌──────────────────▼──────────────────────┐
            │                math                      │
            │  ┌───────────────────────────────────┐  │
            │  │ 平台层: xp_math.hpp / mathdef.hpp │  │
            │  │ (Vector*Base / MatrixBase 定义)   │  │
            │  └───────────────┬───────────────────┘  │
            │  ┌───────────────▼───────────────────┐  │
            │  │ 向量: Vector2/3/4 + VectorFast    │  │
            │  │ 矩阵: Matrix + Quaternion + Angle │  │
            │  │ 几何: BoundingBox/OrientedBBox/   │  │
            │  │       PlaneEq/Polyhedron/...      │  │
            │  │ 统计: EMA/SMA/StatWithRatesOfChange│  │
            │  │ 噪声: PerlinNoise/SimplexNoise    │  │
            │  └───────────────┬───────────────────┘  │
            └──────────────────┼──────────────────────┘
                               │
        ┌──────────────────────┼──────────────────────────┐
        ▼                      ▼                          ▼
   ┌─────────┐          ┌─────────────┐            ┌─────────────┐
   │  moo    │          │  resmgr /   │            │   server    │
   │ 渲染层  │          │  chunk /    │            │ cellapp/... │
   │ (D3D)   │          │  physics    │            │  (无 D3D)   │
   └─────────┘          └─────────────┘            └─────────────┘
```

### 1.3 关键文件清单

| 类别 | 文件 |
|---|---|
| 平台/命名 | xp_math.hpp, mathdef.hpp, math_namespace.hpp, pch.hpp, forward_declarations.hpp |
| 向量 | vector2.hpp, vector3.hpp, vector4.hpp, vector_fast.hpp |
| 矩阵/变换 | matrix.hpp, quat.hpp, blend_transform.hpp, matrix_liason.hpp, angle.hpp, math_extra.hpp |
| 包围/几何 | boundbox.hpp, oriented_bbox.hpp, planeeq.hpp, lineeq.hpp, lineeq3.hpp, polygon.hpp, polyhedron.hpp, convex_hull.hpp, portal2d.hpp, rectt.hpp, range1dt.hpp |
| 统计/插值 | ema.hpp, sma.hpp, linear_lut.hpp, linear_animation.hpp, integrat.hpp, stat_with_rates_of_change.hpp, stat_watcher_creator.hpp |
| 噪声 | perlin_noise.hpp, simplex_noise.hpp |
| 颜色 | colour.hpp |

---

## 2. 平台宏与命名空间

### 2.1 xp_math.hpp —— Base 类的来源

[xp_math.hpp](file:///workspace/src/lib/math/xp_math.hpp) 是 math 库的"平台适配入口"。它的核心职责是定义 `Vector2Base` / `Vector3Base` / `Vector4Base` / `MatrixBase` / `QuaternionBase` 这几个基类——具体类型从哪里来取决于平台：

```cpp
// xp_math.hpp:20-35 (Win32 默认路径)
#if defined(_WIN32)
#include "cstdmf/aligned.hpp"
#ifndef USE_XG_MATH
#include "d3dx9math.h"
#define DIRECTX_MATH
#define EXT_MATH
#include <emmintrin.h>

typedef D3DXMATRIX MatrixBase;
typedef D3DXQUATERNION QuaternionBase;
typedef D3DXVECTOR2 Vector2Base;
typedef D3DXVECTOR3 Vector3Base;
typedef D3DXVECTOR4 Vector4Base;
```

也就是说，**在 Win32 客户端构建中，`Vector3` 实际继承自 `D3DXVECTOR3`，`Matrix` 继承自 `D3DXMATRIX`**，并能直接使用 D3DX 的 SIMD 优化实现（`D3DXVec3Dot` / `D3DXMatrixMultiply` 等，通过 `XPVec3Dot` / `XPMatrixMultiply` 等宏别名暴露）。

```cpp
// xp_math.hpp:41-77
#define XPVec3Length D3DXVec3Length
#define XPVec3Dot D3DXVec3Dot
#define XPVec3Cross D3DXVec3Cross
#define XPVec3Normalize D3DXVec3Normalize
#define XPVec3Lerp D3DXVec3Lerp
#define XPVec3Transform D3DXVec3Transform
// ...
#define XPMatrixIdentity D3DXMatrixIdentity
#define XPMatrixInverse D3DXMatrixInverse
#define XPMatrixMultiply D3DXMatrixMultiply
#define XPMatrixPerspectiveFovLH D3DXMatrixPerspectiveFovLH
#define XPQuaternionSlerp D3DXQuaternionSlerp
// ...
```

在非 Win32 平台（Linux 服务端、PS3、Xbox360 + XG Math），xp_math.hpp 改为自研的 Base 类型（`USE_XG_MATH` 分支或 `#else` 分支），完全不依赖 D3DX。这就是"无 D3D 依赖"的真正含义——**服务端/Linux 构建路径不碰 D3D**。

`Aligned` 基类（来自 cstdmf/aligned.hpp）保证 `Matrix` 等 16 字节对齐，是 SSE `__m128` 操作的前提。

### 2.2 mathdef.hpp —— 全局定义

[mathdef.hpp](file:///workspace/src/lib/math/mathdef.hpp) 提供全局常量与工具函数：

```cpp
// mathdef.hpp:30-31
const float MATH_PI = 3.14159265359f;
const float MATH_2PI = 2.f * 3.14159265359f;

// mathdef.hpp:45-58
inline float DEG_TO_RAD( float angle ) { return angle * MATH_PI / 180.f; }
inline float RAD_TO_DEG( float angle ) { return angle * 180.f / MATH_PI; }
```

还有 `BW_ROUNDF` / `BW_ROUND_TO_INT` 等舍入函数，以及 `Outcode` 类型（用于裁剪，见 §5.1）。它还引入 `BWRandom`，因为部分数学工具需要随机数。

### 2.3 math_namespace.hpp

[math_namespace.hpp](file:///workspace/src/lib/math/math_namespace.hpp) 仅定义两个宏，用于把代码包进 `BW` 命名空间：

```cpp
// math_namespace.hpp:15-16
#define BEGIN_BW_MATH namespace BW {
#define END_BW_MATH }
```

实际引擎中多数类（`Vector3`、`Matrix` 等）位于全局命名空间，仅 `Polyhedron`、`OrientedBBox` 等较新的类放在 `Math` 命名空间下。

### 2.4 forward_declarations.hpp / pch.hpp

- [forward_declarations.hpp](file:///workspace/src/lib/math/forward_declarations.hpp)：前向声明所有数学类，用于减少头文件依赖。
- [pch.hpp](file:///workspace/src/lib/math/pch.hpp)：预编译头，引入 STL 与 cstdmf 常用头。

---

## 3. 向量类

### 3.1 Vector3 —— 最常用的类型

[vector3.hpp](file:///workspace/src/lib/math/vector3.hpp) 定义三维向量，继承 `Vector3Base`：

```cpp
// vector3.hpp:29-85
class Vector3 : public Vector3Base
{
public:
    Vector3();
    Vector3( float a, float b, float c );
    explicit Vector3( const Vector3Base & v );
#ifdef _WIN32
    Vector3( __m128 v4 );
#endif

    void setZero();
    void set( float a, float b, float c );
    void setPitchYaw( float pitchInRadians, float yawInRadians );

    float dotProduct( const Vector3& v ) const;
    void crossProduct( const Vector3& v1, const Vector3& v2 );
    Vector3 crossProduct( const Vector3 & v ) const;
    void lerp( const Vector3 & v1, const Vector3 & v2, float alpha );
    void clamp( const Vector3 & lower, const Vector3 & upper );

    void projectOnto( const Vector3& v1, const Vector3& v2 );
    Vector3 projectOnto( const Vector3 & v ) const;

    INLINE float length() const;
    INLINE float lengthSquared() const;
    INLINE void normalise();
    INLINE Vector3 unitVector() const;

    INLINE void operator += ( const Vector3& v );
    INLINE void operator -= ( const Vector3& v );
    INLINE void operator *= ( float s );
    INLINE void operator /= ( float s );
    INLINE Vector3 operator-() const;

    float yaw() const;
    float pitch() const;
    std::string desc() const;

    static const Vector3 & zero()      { return s_zero; }
private:
    Vector3( int value );   // 防止 Vector3(0) 被解释为 float*
    static Vector3 s_zero;
};
```

**关键 API 摘要**：

| 类别 | 方法 |
|---|---|
| 构造 | `Vector3()`、`Vector3(a,b,c)`、`Vector3(Vector3Base&)`、Win32 上 `Vector3(__m128)` |
| 设置 | `setZero`、`set(a,b,c)`、`setPitchYaw(pitch,yaw)` |
| 代数 | `dotProduct`、`crossProduct`（双参填充/单参返回）、`lerp`、`clamp`、`projectOnto` |
| 模长 | `length`、`lengthSquared`、`normalise`、`unitVector` |
| 角度 | `yaw`、`pitch` |
| 运算符 | `+=`/`-=`/`*=`/`/=`、一元 `-`、二元 `+`/`-`/`*`/`/`、`==`/`!=`/`<`/`>`/`<=`/`>=` |
| 流 | `operator<<` / `operator>>`（文本）、`operator>>` / `operator<<`（`BinaryIStream`/`BinaryOStream`，用于网络序列化） |
| 工具 | `desc()` 返回可读字符串、`zero()` 返回静态零向量、`almostEqual(v1,v2,eps)` 全局函数 |

一个值得注意的细节：构造函数 `Vector3(int)` 被声明为 `private` 且不实现，用于在编译期阻止 `Vector3 v(0);` 这种把 `0` 当 `float*` 用的隐式转换 bug。

`BinaryIStream` / `BinaryOStream` 的重载让 `Vector3` 可直接通过 cstdmf 的二进制流序列化，是网络协议传递位置/速度的基础。

### 3.2 Vector2 / Vector4

[vector2.hpp](file:///workspace/src/lib/math/vector2.hpp) 与 [vector4.hpp](file:///workspace/src/lib/math/vector4.hpp) 提供二维与四维向量，API 与 `Vector3` 类似但适配维度：

`Vector2` 关键差异：
- `crossProduct` 返回 `float`（二维叉积的标量结果）。
- 无 `pitch`/`yaw`。

`Vector4` 关键差异：
- 多了 `parallelMultiply`（逐分量相乘，用于颜色混合）。
- 多了 `calculateOutcode`（用于裁剪管线）。
- 可从 `Vector3 + w` 构造：`Vector4(const Vector3& v, float w)`。

### 3.3 VectorFast —— SIMD 显式变体

[vector_fast.hpp](file:///workspace/src/lib/math/vector_fast.hpp) 定义 `VectorFastBase`，强制 16 字节对齐并直接持 `__m128`：

```cpp
// vector_fast.hpp:28-78
class VectorFastBase : public Aligned
{
public:
    VectorFastBase();
    VectorFastBase( const VectorFastBase& v );
    VectorFastBase( __m128 v );
    VectorFastBase( float a, float b, float c, float d = 1.f );
    explicit VectorFastBase( const Vector3& v );
    explicit VectorFastBase( const Vector4& v );

    void getVector3( Vector3& v ) const;
    void castToInt( int* i ) const;
    void setZero();
    void saturate();

    VectorFastBase& operator = ( const VectorFastBase& v );
    VectorFastBase& operator = ( __m128 v );
    VectorFastBase& operator += ( const VectorFastBase& v );
    // ...
private:
    VectorFastBase( int value );
public:
    union
    {
        __m128                  v4;
        float                   f[4];
        struct { float x, y, z, w; };
    };
};
```

`union` 让同一块 16 字节内存可按 `__m128`、`float[4]`、或具名 `x/y/z/w` 访问。`VectorFast` 用于渲染热路径（顶点变换、批量矩阵乘），通过显式 SIMD 比依赖编译器自动向量化更可控。注意它继承 `Aligned`，因此只能用对齐分配器（如 cstdmf 的 `aallocator`）存放进容器。

### 3.4 向量类关系图

```
        Aligned (cstdmf)
           ▲
           │
    VectorFastBase ──union──> __m128 / float[4] / {x,y,z,w}
           ▲
           │
       VectorFast

    ─────────────────────────────────────────

    Vector2Base (D3DXVECTOR2 / 自研)
           ▲
           │
        Vector2

    Vector3Base (D3DXVECTOR3 / 自研)
           ▲
           │
        Vector3  ◄── 可互转 ──► VectorFastBase

    Vector4Base (D3DXVECTOR4 / 自研)
           ▲
           │
        Vector4  ◄── 可从 Vector3+w 构造
```

---

## 4. 矩阵与变换

### 4.1 Matrix —— 4×4 变换矩阵

[matrix.hpp](file:///workspace/src/lib/math/matrix.hpp) 定义 4×4 矩阵，继承 `MatrixBase`（Win32 上即 `D3DXMATRIX`）：

```cpp
// matrix.hpp:33-126
class Matrix : public MatrixBase
{
public:
    Matrix();
    INLINE Matrix( const Vector4& v0, const Vector4& v1, const Vector4& v2, const Vector4& v3 );

    INLINE void setZero();
    INLINE void setIdentity();

    void setScale( const float x, const float y, const float z );
    void setScale( const Vector3 & scale );
    void setTranslate( const float x, const float y, const float z );
    void setTranslate( const Vector3 & pos );
    void setRotateX( const float angle );
    void setRotateY( const float angle );
    void setRotateZ( const float angle );
    void setRotate( const Quaternion & q );
    void setRotate( float yaw, float pitch, float roll );
    void setRotateInverse( float yaw, float pitch, float roll );

    void multiply( const Matrix& m1, const Matrix& m2 );
    void preMultiply( const Matrix& m );
    void postMultiply( const Matrix& m );

    void invertOrthonormal( const Matrix& m );
    void invertOrthonormal();
    bool invert( const Matrix& m );
    bool invert();
    float getDeterminant() const;

    void transpose( const Matrix & m );
    void transpose();

    void lookAt( const Vector3& position, const Vector3& direction, const Vector3& up );

    float& operator ()( uint32 column, uint32 row );
    float  operator ()( uint32 column, uint32 row ) const;

    Vector3 applyPoint( const Vector3& v2 ) const;
    void    applyPoint( Vector3&v1, const Vector3& v2) const;
    void    applyPoint( Vector4&v1, const Vector3& v2) const;
    void    applyPoint( Vector4&v1, const Vector4& v2) const;
    INLINE Vector3 applyVector( const Vector3& v2 ) const;
    INLINE void    applyVector( Vector3& v1, const Vector3& v2 ) const;

    const Vector3 & applyToUnitAxisVector( int axis ) const;
    const Vector3 & applyToOrigin() const;

    INLINE Vector3 & operator []( int i );
    INLINE const Vector3 & operator []( int i ) const;
    INLINE void         row( int i, const Vector4 & value );
    INLINE const Vector4& row( int i ) const;
    Vector4 column( int i ) const;
    void column( int i, const Vector4 & v );

    void preRotateX/Y/Z(const float angle);
    void preTranslateBy(const Vector3 & v);
    void postRotateX/Y/Z(const float angle);
    void postTranslateBy(const Vector3 & v);

    bool isMirrored() const;

    void orthogonalProjection( float w, float h, float zn, float zf );
    void perspectiveProjection( float fov, float aspectRatio, float nearPlane, float farPlane );

    void translation( const Vector3& v );

    float yaw() const;
    float pitch() const;
    float roll() const;

    static const Matrix identity;
};
```

**关键 API 摘要**：

| 类别 | 方法 |
|---|---|
| 构造/设置 | `setZero`、`setIdentity`、4 个 `Vector4` 构造 |
| 缩放/平移/旋转 | `setScale`、`setTranslate`、`setRotateX/Y/Z`、`setRotate(Quaternion)`、`setRotate(yaw,pitch,roll)` |
| 矩阵乘法 | `multiply(m1,m2)`、`preMultiply(m)`、`postMultiply(m)` |
| 求逆/转置 | `invert`（通用）、`invertOrthonormal`（正交快路径）、`transpose`、`getDeterminant` |
| 视图/投影 | `lookAt`、`orthogonalProjection`、`perspectiveProjection` |
| 应用变换 | `applyPoint`（点，含平移）、`applyVector`（向量，不含平移）、`applyToUnitAxisVector`、`applyToOrigin` |
| 元素访问 | `operator()(col,row)`、`operator[](i)`（行访问）、`row`/`column` |
| 链式操作 | `preRotateX/Y/Z`、`preTranslateBy`、`postRotateX/Y/Z`、`postTranslateBy` |
| 查询 | `isMirrored`、`yaw`/`pitch`/`roll` |

`typedef Matrix Matrix34; typedef Matrix Matrix44;` 表明 BigWorld 用同一个 4×4 矩阵同时表示 3×4 仿射变换与完整 4×4 投影矩阵。

`applyPoint` 与 `applyVector` 的区别很重要：点变换会应用平移分量（w=1），向量变换不会（w=0）。这在法线、方向向量变换时关键。

### 4.2 Quaternion —— 四元数

[quat.hpp](file:///workspace/src/lib/math/quat.hpp) 定义四元数，继承 `QuaternionBase`（Win32 上即 `D3DXQUATERNION`）：

```cpp
// quat.hpp:30-63
class Quaternion : public QuaternionBase
{
public:
    Quaternion();
    Quaternion( const Matrix &m );
    Quaternion( float x, float y, float z, float w );
    Quaternion( const Vector3 &v, float w );

    void setZero();
    void set( float x, float y, float z, float w );
    void set( const Vector3 &v, float w );

    void fromAngleAxis( float angle, const Vector3 &axis );
    void fromMatrix( const Matrix &m );

    void normalise();
    void invert();
    void minimise();

    void slerp( const Quaternion& qStart, const Quaternion &qEnd, float t );

    void multiply( const Quaternion& q1, const Quaternion& q2 );
    void preMultiply( const Quaternion& q );
    void postMultiply( const Quaternion& q );

    float dotProduct( const Quaternion& q ) const;
    float length() const;
    float lengthSquared() const;
};

Quaternion  operator *( const Quaternion& q1, const Quaternion& q2 );
bool        operator ==( const Quaternion& q1, const Quaternion& q2 );
```

**关键 API 摘要**：

| 类别 | 方法 |
|---|---|
| 构造 | 默认、从 `Matrix`、`(x,y,z,w)`、`(Vector3, w)` |
| 转换 | `fromAngleAxis`（轴角→四元数）、`fromMatrix`（矩阵→四元数） |
| 操作 | `normalise`、`invert`、`minimise`（取最短弧） |
| 插值 | `slerp`（球面线性插值，动画混合核心） |
| 乘法 | `multiply`、`preMultiply`、`postMultiply`、`operator*` |
| 查询 | `dotProduct`、`length`、`lengthSquared` |

`slerp` 是角色动画在不同动作间平滑过渡的基础。`minimise` 处理四元数双重覆盖问题（q 和 -q 表示同一旋转，选更短路径）。

### 4.3 BlendTransform —— 变换混合

[blend_transform.hpp](file:///workspace/src/lib/math/blend_transform.hpp) 把旋转/缩放/平移分解存储，专用于动画混合：

```cpp
// blend_transform.hpp:24-90
class BlendTransform
{
private:
    Quaternion  rotate_;
    Vector3     scale_;
    Vector3     translate_;
public:
    const Quaternion& rotation() const;
    const Vector3& scaling() const;
    const Vector3& translation() const;
    void translation( const Vector3 & translation );

    void init( const Matrix& ma );
    explicit BlendTransform( const Matrix& m );

    inline BlendTransform() :
        rotate_( 0, 0, 0, 1 ),
        scale_( 1, 1, 1 ),
        translate_( 0, 0, 0 )
    {
    }

    BlendTransform( const Quaternion & irotate, const Vector3 & iscale, const Vector3 & itranslate );

    inline void normaliseRotation()
    {
        rotate_.normalise();
    }
    // ... blend() 等
};
```

**为什么需要 BlendTransform**：直接对 `Matrix` 做线性插值会破坏正交性（旋转矩阵的线性插值不再是旋转矩阵）。`BlendTransform` 把变换分解为四元数（旋转）+ 向量（缩放）+ 向量（平移），三者可独立插值——四元数用 `slerp`，向量用 `lerp`——最后再合成回 `Matrix`。这是骨骼动画的标准做法。

### 4.4 matrix_liason / angle / math_extra

- [matrix_liason.hpp](file:///workspace/src/lib/math/matrix_liason.hpp)：`MatrixLiason` 接口，让非 `Matrix` 类（如物理引擎自有矩阵）能暴露统一的矩阵视图。
- [angle.hpp](file:///workspace/src/lib/math/angle.hpp)：`Angle` 类，把角度封装为类型，区分弧度/度，防止单位混淆。
- [math_extra.hpp](file:///workspace/src/lib/math/math_extra.hpp)：杂项数学工具函数（如 `almostEqual`、夹角计算等）。

### 4.5 矩阵/变换类关系图

```
      MatrixBase (D3DXMATRIX / 自研)
           ▲
           │
        Matrix ──────┬────► applyPoint/applyVector (变换 Vector3/4)
           ▲         │
           │         └────► setRotate(Quaternion)
           │
      Quaternion ◄────── fromMatrix(Matrix)
           ▲
           │
     QuaternionBase (D3DXQUATERNION / 自研)

      BlendTransform
        ├─ Quaternion rotate_
        ├─ Vector3    scale_
        └─ Vector3    translate_
           │
           └── init(Matrix) / 输出 Matrix (合成)
```

---

## 5. 包围与几何

### 5.1 BoundingBox —— 轴对齐包围盒

[boundbox.hpp](file:///workspace/src/lib/math/boundbox.hpp) 定义轴对齐包围盒（AABB），是引擎中最常用的包围体：

```cpp
// boundbox.hpp:26-80
class BoundingBox
{
public:
    BoundingBox();
    BoundingBox( const Vector3 & min, const Vector3 & max );

    bool operator==( const BoundingBox& bb ) const;
    bool operator!=( const BoundingBox& bb ) const;

    const Vector3 & minBounds() const;
    const Vector3 & maxBounds() const;
    void setBounds( const Vector3 & min, const Vector3 & max );

    float width() const;
    float height() const;
    float depth() const;

    void addYBounds( float y );
    void addBounds( const Vector3 & v );
    void addBounds( const BoundingBox & bb );
    void expandSymmetrically( float dx, float dy, float dz );
    void expandSymmetrically( const Vector3 & v );
    void calculateOutcode( const Matrix & m ) const;

    Outcode outcode() const;
    Outcode combinedOutcode() const;
    void outcode( Outcode oc );
    void combinedOutcode( Outcode oc );

    void transformBy( const Matrix & transform );

    bool intersects( const BoundingBox & box ) const;
    bool intersects( const Vector3 & v ) const;
    bool intersects( const Vector3 & v, float bias ) const;
    bool intersectsRay( const Vector3 & origin, const Vector3 & dir ) const;
    bool intersectsLine( const Vector3 & origin, const Vector3 & dest ) const;

    bool clip( Vector3 & start, Vector3 & extent, float bloat = 0.f ) const;

    float distance( const Vector3& point ) const;

    INLINE Vector3 centre() const;
    INLINE bool insideOut() const;

private:
    Vector3 min_;
    Vector3 max_;
    mutable Outcode oc_;
    mutable Outcode combinedOc_;
public:
    static const BoundingBox s_insideOut_;
};
```

**关键 API 摘要**：

| 类别 | 方法 |
|---|---|
| 构造/设置 | 默认、`(min,max)`、`setBounds` |
| 尺寸 | `width`/`height`/`depth`/`centre` |
| 扩展 | `addBounds(Vector3)`、`addBounds(BoundingBox)`、`addYBounds`、`expandSymmetrically` |
| 裁剪 | `calculateOutcode(Matrix)`、`outcode`/`combinedOutcode`（位掩码标记盒在视锥各面的外侧） |
| 变换 | `transformBy(Matrix)`（注意 AABB 变换后仍是 AABB，会膨胀） |
| 相交测试 | `intersects(BoundingBox)`、`intersects(Vector3)`、`intersectsRay`、`intersectsLine` |
| 裁剪 | `clip(start,extent)`（Cohen-Sutherland 风格线段裁剪） |
| 距离 | `distance(Vector3)`（点到盒最近距离） |
| 特殊状态 | `insideOut()`（min>max，表示"空"或"未初始化"盒）、`s_insideOut_` 静态实例 |

`Outcode` 是位掩码，每位代表盒在视锥某一裁剪面外侧。`calculateOutcode` 把盒的 8 个顶点用矩阵变换到裁剪空间并计算联合 outcode，是粗粒度视锥剔除的核心——`combinedOc_ == 0` 表示盒完全在视锥内，无需进一步处理。

### 5.2 OrientedBBox —— 有向包围盒

[oriented_bbox.hpp](file:///workspace/src/lib/math/oriented_bbox.hpp) 定义有向包围盒，位于 `Math` 命名空间，继承 `Polyhedron`：

```cpp
// oriented_bbox.hpp:23-56
namespace Math
{
class OrientedBBox : public Polyhedron
{
public:
    OrientedBBox();
    OrientedBBox( const BoundingBox& bb, const Matrix& matrix );
    void create( const BoundingBox& bb, const Matrix& matrix );

    unsigned int numPoints() const;
    unsigned int numFaces() const;
    unsigned int numEdges() const;
    Vector3 point( unsigned int i ) const;
    Face face( unsigned int i ) const;
    Edge edge( unsigned int i ) const;
private:
    std::vector<Vector3> points_;
    std::vector<Face> faces_;
    std::vector<Edge> edges_;
};
}
```

`OrientedBBox` 把 `BoundingBox` 用 `Matrix` 变换后得到 8 个顶点、6 个面、12 条边，作为 `Polyhedron` 参与更精确的相交测试。当 AABB 相交测试不够精确（如旋转物体的碰撞）时使用。

### 5.3 PlaneEq —— 平面方程

[planeeq.hpp](file:///workspace/src/lib/math/planeeq.hpp) 定义 3D 平面，存为法向量 + 距离 `d`（`n·p + d = 0`）：

```cpp
// planeeq.hpp:28-78
class PlaneEq
{
public:
    enum ShouldNormalise
    {
        SHOULD_NORMALISE,
        SHOULD_NOT_NORMALISE
    };

    PlaneEq();
    PlaneEq( const Vector3 & normal, const float d );
    PlaneEq( const Vector3 & point, const Vector3 & normal );
    PlaneEq( const Vector3 & v0, const Vector3 & v1, const Vector3 & v2,
        ShouldNormalise normalise = SHOULD_NORMALISE );

    INLINE void init( const Vector3 & p0, const Vector3 & p1, const Vector3 & p2,
        ShouldNormalise normalise = SHOULD_NORMALISE );
    INLINE void init( const Vector3 & point, const Vector3 & normal );

    float distanceTo( const Vector3 & point ) const;
    bool isInFrontOf( const Vector3 & point ) const;
    bool isInFrontOfExact( const Vector3 & point ) const;
    float y( float x, float z ) const;

    Vector3 intersectRay( const Vector3 & source, const Vector3 & dir ) const;
    INLINE float intersectRayHalf( const Vector3 & source, float normalDotDir ) const;
    INLINE float intersectRayHalfNoCheck( const Vector3 & source, float oneOverNormalDotDir ) const;

    LineEq intersect( const PlaneEq & slice ) const;

    void basis( Vector3 & xdir, Vector3 & ydir ) const;
    Vector3 param( const Vector2 & param ) const;
    Vector2 project( const Vector3 & point ) const;

    const Vector3 & normal() const;
    float d() const;

    void setHidden();
    void setAlwaysVisible();
private:
    Vector3 normal_;
    float d_;
};
```

**关键 API 摘要**：

| 类别 | 方法 |
|---|---|
| 构造 | `(normal,d)`、`(point,normal)`、`(v0,v1,v2)`（三点定面，可选归一化） |
| 距离/判断 | `distanceTo`、`isInFrontOf`、`isInFrontOfExact`、`y(x,z)`（地形高度查询） |
| 射线相交 | `intersectRay`、`intersectRayHalf`（已知 `n·dir`）、`intersectRayHalfNoCheck`（已知 `1/(n·dir)`，最快） |
| 平面相交 | `intersect(PlaneEq)` 返回 `LineEq` |
| 参数化 | `basis`（求平面内两个正交基向量）、`param(Vector2)`（参数→3D 点）、`project`（3D 点→2D 参数） |
| 可见性 | `setHidden` / `setAlwaysVisible`（用于地块 portals） |

`intersectRayHalfNoCheck` 是热路径优化——调用方预先算好 `1/(n·dir)`，避免每次相交测试都做除法。`setHidden`/`setAlwaysVisible` 通过把 `d_` 设为特殊值实现，用于地块门户系统的快速剔除。

### 5.4 LineEq / LineEq3 / Polygon / Polyhedron / ConvexHull / Portal2D / RectT / Range1dt

- [lineeq.hpp](file:///workspace/src/lib/math/lineeq.hpp)：2D 直线方程，用于平面相交结果。
- [lineeq3.hpp](file:///workspace/src/lib/math/lineeq3.hpp)：3D 直线。
- [polygon.hpp](file:///workspace/src/lib/math/polygon.hpp)：多边形（凸/凹），含面积、重心、包含测试。
- [polyhedron.hpp](file:///workspace/src/lib/math/polyhedron.hpp)：多面体抽象基类，定义 `Face`/`Edge` 内嵌类与相交测试接口，`OrientedBBox` 继承它：

```cpp
// polyhedron.hpp:27-99
namespace Math
{
class Polyhedron
{
public:
    class Face { /* 持有 owner_ + idxs_ + normal_ */ };
    class Edge { /* 持有 owner_ + idx0_ + idx1_ */ };

    virtual unsigned int numPoints() const = 0;
    virtual unsigned int numFaces() const = 0;
    virtual unsigned int numEdges() const = 0;
    virtual Vector3 point( unsigned int i ) const = 0;
    virtual Face face( unsigned int i ) const = 0;
    virtual Edge edge( unsigned int i ) const = 0;
    virtual bool intersects( const Polyhedron& other ) const;
    // ...
};
}
```

- [convex_hull.hpp](file:///workspace/src/lib/math/convex_hull.hpp)：凸包计算。
- [portal2d.hpp](file:///workspace/src/lib/math/portal2d.hpp)：2D 门户（用于地块间可见性传递）。
- [rectt.hpp](file:///workspace/src/lib/math/rectt.hpp)：模板化矩形 `RectT<T>`，常用于 UI 与纹理区域。
- [range1dt.hpp](file:///workspace/src/lib/math/range1dt.hpp)：1D 区间 `Range1D<T>`，表示 `[min,max]`。

### 5.5 几何类关系图

```
   BoundingBox (AABB)
       │
       │ transformBy(Matrix) 后用作输入
       ▼
   OrientedBBox ──is-a──► Polyhedron (抽象)
                              ▲
                              │
                          ConvexHull (?)  (参见 convex_hull.hpp)

   PlaneEq ──intersect──► LineEq (2D)
       │
       │ intersectRay
       ▼
   Vector3 (交点)

   Polygon (2D 多边形)      Portal2D (2D 门户)
       │                        │
       └────────┬───────────────┘
                ▼
            RectT<T> / Range1D<T> (基础区间/矩形)
```

---

## 6. 统计与插值

### 6.1 EMA —— 指数移动平均

[ema.hpp](file:///workspace/src/lib/math/ema.hpp) 实现指数加权移动平均，公式为 `avg(n) = (1-bias)*sample(n) + bias*avg(n-1)`：

```cpp
// ema.hpp:24-63
class EMA
{
public:
    explicit EMA( float bias, float initial=0.f ):
        bias_( bias ),
        average_( initial )
    {}

    void sample( float value )
    {
        average_ = (1.f - bias_) * value + bias_ * average_;
    }

    float average() const   { return average_; }

    static float calculateBiasFromNumSamples( float numSamples,
            float weighting = 0.95f );

private:
    float bias_;
    float average_;
};
```

`calculateBiasFromNumSamples` 根据期望的"等效样本数"反算 bias——例如想让平均等效于最近 30 个样本的加权，可调用此函数。

`AccumulatingEMA<T>` 是 EMA 的累加变体，先把样本累加到 `accum_`，周期性 `sample()` 时把累加值送入 EMA 并清零，适合"事件计数型"统计（如每秒收到的字节数）：

```cpp
// ema.hpp:69-129
template< typename TYPE >
class AccumulatingEMA
{
public:
    AccumulatingEMA( float bias, float initialAverage=0.f,
            TYPE initialValue=TYPE() ):
        average_( bias, initialAverage ),
        accum_( initialValue )
    { }

    TYPE & value()              { return accum_; }
    const TYPE & value() const  { return accum_; }
    float average() const       { return average_.average(); }

    void sample( bool shouldReset=true )
    {
        average_.sample( float( accum_ ) );
        if (shouldReset)
        {
            accum_ = Type();
        }
    }
private:
    EMA     average_;
    TYPE    accum_;
};
```

### 6.2 SMA —— 简单移动平均

[sma.hpp](file:///workspace/src/lib/math/sma.hpp) 实现简单移动平均（窗口内等权）：

```cpp
// sma.hpp:23-51
template<class T> class SMA
{
public:
    SMA(int period);
    void append(T value);
    T average() const;
    T min() const;
    T max() const;
    int period() const         { return period_; }
    int count() const          { return count_; }
    void clear();
private:
    int     period_;
    T*      samples_;
    T       total_;
    int     count_;
    int     pos_;
};
```

用环形缓冲实现，`append` 时减去将被覆盖的旧值、加上新值、更新 `total_`，`average()` 直接 `total_/count_`，O(1) 复杂度。`min`/`max` 需 O(count) 扫描。断言限制 period 在 (2, 1000)。

### 6.3 StatWithRatesOfChange —— 带变化率统计

[stat_with_rates_of_change.hpp](file:///workspace/src/lib/math/stat_with_rates_of_change.hpp) 是 cellapp 带宽统计等场景的核心工具，它维护一个累计值 `total_` 并用多个不同时间窗口的 EMA 跟踪其变化率：

```cpp
// stat_with_rates_of_change.hpp:27-70
template < class TYPE >
class StatWithRatesOfChange
{
public:
    StatWithRatesOfChange();
    void monitorRateOfChange( float numSamples );
    void tick( double deltaTime );

    void operator++()        { this->add( 1 ); }
    void operator++( int )   { this->add( 1 ); }
    void operator+=( TYPE value )   { this->add( value ); }
    void operator-=( TYPE value )   { this->subtract( value ); }

    TYPE total() const;
    void setTotal( TYPE total );

    double getRateOfChange( int index = 0 ) const;

    // 为 Watcher 提供的类型化访问器（因模板成员函数不能直接做 watcher）
    double getRateOfChange0() const { return this->getRateOfChange( 0 ); }
    double getRateOfChange1() const { return this->getRateOfChange( 1 ); }
    double getRateOfChange2() const { return this->getRateOfChange( 2 ); }
    double getRateOfChange3() const { return this->getRateOfChange( 3 ); }
    double getRateOfChange4() const { return this->getRateOfChange( 4 ); }
private:
    void add( TYPE value );
    void subtract( TYPE value );
    TYPE total_;
    TYPE prevTotal_;
    typedef std::vector< EMA > Averages;
    Averages averages_;
};
```

`tick(deltaTime)` 每帧调用，计算 `total_ - prevTotal_` 作为本帧增量，送入所有 EMA，更新 `prevTotal_`。`getRateOfChange(index)` 返回第 `index` 个 EMA 的当前平均值，对应不同时间尺度的"每秒变化量"。

`getRateOfChange0..4` 这些硬编码访问器是为了规避"模板成员函数不能直接注册为 watcher"的限制——`MemberWatcher` 需要具体类型的成员函数指针。`IntrusiveStatWithRatesOfChange` 是其侵入式链表版本，便于把所有统计对象串起来统一上报。

### 6.4 LinearLUT —— 线性查找表

[linear_lut.hpp](file:///workspace/src/lib/math/linear_lut.hpp) 实现可插值的查找表，支持 4 种边界条件：

```cpp
// linear_lut.hpp:23-52
class LinearLUT
{
public:
    enum BoundaryCondition
    {
        BC_ZERO,                // 越界返回 0
        BC_CONSTANT_EXTEND,     // 越界取端点值
        BC_WRAP,                // 越界回绕
        BC_LINEAR_EXTEND        // 越界线性外推
    };

    LinearLUT();
    float operator()(float x) const;
    void data(std::vector<Vector2> const &d);
    std::vector<Vector2> const &data() const;
    void lowerBoundaryCondition(BoundaryCondition bc);
    void upperBoundaryCondition(BoundaryCondition bc);
private:
    std::vector<Vector2>            data_;
    BoundaryCondition               lowerBC_;
    BoundaryCondition               upperBC_;
    mutable size_t                  cachedPos_;
};
```

数据是 `Vector2` 数组（x=输入、y=输出），`operator()` 做线性插值。`cachedPos_` 缓存上次查询位置，假设查询通常顺序递增，可避免二分查找。用于噪声权重曲线、动画曲线等。

### 6.5 LinearAnimation / Integrat

- [linear_animation.hpp](file:///workspace/src/lib/math/linear_animation.hpp)：基于关键帧的线性动画曲线，时间→值的插值。
- [integrat.hpp](file:///workspace/src/lib/math/integrat.hpp)：数值积分工具（梯形/辛普森等），用于物理与统计。

### 6.6 stat_watcher_creator

[stat_watcher_creator.hpp](file:///workspace/src/lib/math/stat_watcher_creator.hpp) 是辅助模板，把 `StatWithRatesOfChange` 的多个 EMA 自动注册为 watcher 目录，省去手写 `getRateOfChange0..4` 的样板代码。

### 6.7 统计类关系图

```
   EMA (单窗口指数平均)
     ▲
     │ 持有多个
     │
   StatWithRatesOfChange<TYPE>
     ├─ total_ / prevTotal_
     └─ vector<EMA> averages_
     ▲
     │
   IntrusiveStatWithRatesOfChange<TYPE>  (加侵入式链表)

   AccumulatingEMA<TYPE>
     ├─ EMA average_
     └─ TYPE accum_

   SMA<T> (环形缓冲 + 等权平均)

   LinearLUT (插值查找表, 持 vector<Vector2>)
```

---

## 7. 噪声

### 7.1 PerlinNoise

[perlin_noise.hpp](file:///workspace/src/lib/math/perlin_noise.hpp) 实现 Ken Perlin 的经典噪声：

```cpp
// perlin_noise.hpp:26-43
class PerlinNoise
{
public:
    /* coherent noise function over 1, 2 or 3 dimensions */
    /* (copyright Ken Perlin) */
    void init(void);
    float noise1( float arg );
    float noise2( const Vector2 & vec );
    float noise3( float vec[3] );
    void normalise2( float v[2] );
    void normalise3( float v[3] );

    /* MF's original extensions */
    float sumHarmonics2( const Vector2 & vec, float persistence,
                         float frequency, float nIterations );
private:
};
```

`noise1/2/3` 是 Perlin 原始实现（注释标注 copyright Ken Perlin）。`sumHarmonics2` 是 BigWorld 的扩展——把多个不同频率的 noise 叠加（分形布朗运动 fBm），参数 `persistence` 控制高频贡献衰减、`frequency` 是基频、`nIterations` 是叠加层数。用于地形纹理分布、云层生成等。

### 7.2 SimplexNoise

[simplex_noise.hpp](file:///workspace/src/lib/math/simplex_noise.hpp) 实现 Perlin 改进的 Simplex 噪声，基于 Stefan Gustavson 的实现：

```cpp
// simplex_noise.hpp:37-69
class SimplexNoise
{
public:
    struct Octave
    {
        float       waveLength_;    // 噪声波长
        float       weight_;        // 该层权重
        int         seed_;          // 种子，256 后重复
        Octave();
        Octave( float wavelength, float weight, int seed = 0 );
        float       frequency_;     // 内部设置
    };

    typedef std::vector< Octave > OctaveVec;

    static float generate( float x, float y, int seed );

    SimplexNoise();
    float operator()( float x, float y ) const;
    bool operator==( const SimplexNoise & other ) const;
    bool operator!=( const SimplexNoise & other ) const;

    void octaves( OctaveVec const &oct );
    OctaveVec const &octaves() const;
private:
    std::vector<Octave>             octaves_;
    float                           totalWeight_;
};
```

`SimplexNoise` 比 `PerlinNoise` 更现代：
- 静态 `generate(x,y,seed)` 一次性生成 [0,1] 范围噪声。
- 实例版支持多八度（`Octave` 数组），每层有独立波长/权重/种子，`operator()` 返回加权叠加结果。
- `operator==` 让噪声配置可比较，便于序列化与缓存。

Simplex 噪声计算复杂度低于 Perlin（O(n²) vs O(2ⁿ)），且无方向性伪影，是新一代地形/纹理生成的首选。

---

## 8. 颜色

[colour.hpp](file:///workspace/src/lib/math/colour.hpp) 提供 `Colour` 命名空间下的颜色工具函数，主要是 `Vector3`/`Vector4` 与 `uint32`（ARGB）之间的转换：

```cpp
// colour.hpp:20-79
namespace Colour
{
inline uint32 getUint32( int r, int g, int b, int a = 0 );          // 带 clamp
inline uint32 getUint32NoClamp( int r, int g, int b, int a = 0 );   // 不 clamp
inline uint32 getUint32NoClamp( int* c );
inline uint32 getUint32( const Vector3 &colour, int a );
inline uint32 getUint32NoClamp( const Vector3 &colour, int a );
inline uint32 getUint32( const Vector3 &colour );
// ...
}
```

`getUint32` 把 `(r,g,b,a)` 打包为 `(a<<24)|(r<<16)|(g<<8)|b`，自动 clamp 到 [0,255]。`Vector3` 版本把三分量当作 r/g/b。这套函数用于把浮点颜色（光照计算结果）转为 GPU 可用的 32 位颜色。

---

## 9. 真实代码示例

### 9.1 向量与矩阵基础运算

```cpp
#include "math/vector3.hpp"
#include "math/matrix.hpp"
#include "math/quat.hpp"

// 角色面向某点
void faceTarget( Vector3 & pos, Vector3 & target, Matrix & out )
{
    Vector3 dir = target - pos;
    dir.normalise();

    Matrix m;
    m.setIdentity();
    m.lookAt( pos, dir, Vector3( 0, 1, 0 ) );  // 位置、方向、上方向
    out = m;
}

// 旋转插值（动画过渡）
Quaternion blendRot( const Quaternion & from, const Quaternion & to, float t )
{
    Quaternion result;
    result.slerp( from, to, t );
    return result;
}

// 把点变换到世界空间
Vector3 toWorld( const Matrix & world, const Vector3 & local )
{
    return world.applyPoint( local );
}

// 把方向向量变换（不含平移）
Vector3 dirToWorld( const Matrix & world, const Vector3 & localDir )
{
    return world.applyVector( localDir );
}
```

### 9.2 包围盒与相交测试

```cpp
#include "math/boundbox.hpp"
#include "math/planeeq.hpp"

// 视锥剔除粗判
bool isVisible( const BoundingBox & box, const Matrix & viewProj )
{
    box.calculateOutcode( viewProj );
    Outcode oc = box.combinedOutcode();
    return oc == 0;  // 0 表示完全在视锥内或与某面相交
}

// 射线与 AABB 相交（鼠标拾取）
bool pickObject( const BoundingBox & box,
    const Vector3 & rayOrigin, const Vector3 & rayDir )
{
    return box.intersectsRay( rayOrigin, rayDir );
}

// 平面与射线相交（地形点击）
Vector3 pickGround( const PlaneEq & ground,
    const Vector3 & rayOrigin, const Vector3 & rayDir )
{
    return ground.intersectRay( rayOrigin, rayDir );
}
```

### 9.3 统计平均（带宽监控）

```cpp
#include "math/stat_with_rates_of_change.hpp"

// cellapp 收发字节数统计
StatWithRatesOfChange<uint64> bytesIn_;
StatWithRatesOfChange<uint64> bytesOut_;

// 构造时配置多个时间窗口
bytesIn_.monitorRateOfChange( 30 );   // 近 30 帧
bytesIn_.monitorRateOfChange( 300 );  // 近 300 帧
bytesIn_.monitorRateOfChange( 3000 ); // 近 3000 帧

// 每收到包
bytesIn_ += packetSize;

// 每帧
bytesIn_.tick( deltaTime );

// 查询每秒速率
double rate30  = bytesIn_.getRateOfChange(0);  // 短期
double rate300 = bytesIn_.getRateOfChange(1);  // 中期
double rate3000 = bytesIn_.getRateOfChange(2); // 长期

// 注册到 watcher 供远程查看
MF_WATCH( "network/bytesInRate30",
        *this,
        &CellApp::bytesInRate30 );  // bytesInRate30 内部调 getRateOfChange0()
```

### 9.4 噪声生成地形纹理密度

```cpp
#include "math/simplex_noise.hpp"

// 多八度 simplex 噪声决定草地密度
SimplexNoise grassNoise;
SimplexNoise::OctaveVec octs;
octs.push_back( SimplexNoise::Octave( 50.f,  1.0f, 1 ) );  // 大块
octs.push_back( SimplexNoise::Octave( 20.f,  0.5f, 2 ) );  // 中块
octs.push_back( SimplexNoise::Octave(  5.f,  0.25f, 3 ) ); // 细节
grassNoise.octaves( octs );

float grassDensity( float x, float z )
{
    return grassNoise( x, z );  // [0,1]
}
```

### 9.5 BlendTransform 动画混合

```cpp
#include "math/blend_transform.hpp"

// 两个关键帧变换混合
BlendTransform btA( matrixA );  // 从矩阵分解为 quat + scale + translate
BlendTransform btB( matrixB );

// t 从 0→1 时从 A 平滑过渡到 B
BlendTransform blended;
// blended.blend( btA, btB, t );  // 内部对 quat 做 slerp，对 scale/translate 做 lerp
Matrix result;
// blended.output( result );  // 合成回矩阵供渲染
```

（`blend` / `output` 方法的具体签名参见 [blend_transform.hpp](file:///workspace/src/lib/math/blend_transform.hpp) 完整定义。）

---

## 10. 设计要点小结

1. **Base 类策略**：通过 `xp_math.hpp` 把平台差异封装在 Base 类层，业务类型 `Vector3`/`Matrix` 等保持统一接口。Win32 客户端借 D3DX9 获得 SIMD 加速，服务端/Linux 用自研实现，二者代码级兼容。

2. **INLINE 与 .ipp**：大量方法声明为 `INLINE`（cstdmf 的跨平台 inline 宏），实现放在同名的 `.ipp` 文件中，由 `#ifdef CODE_INLINE` 控制 include 时机——开启时可整体内联，关闭时分离编译减少代码膨胀。

3. **private 构造防误用**：`Vector3(int)`、`Vector4(int)` 等私有构造阻止 `Vector3 v(0)` 这类隐式转换 bug，是引擎代码的常见防御模式。

4. **几何与裁剪协同**：`BoundingBox` 的 `Outcode` 机制与渲染管线的视锥剔除紧密协同；`PlaneEq` 的 `setHidden`/`setAlwaysVisible` 与 chunk portal 系统协同。math 库不只是纯数学，还嵌入了引擎特定的优化钩子。

5. **统计与 watcher 深度集成**：`StatWithRatesOfChange` 的 `getRateOfChange0..4` 硬编码访问器反映了 BigWorld "运行时可观测"的工程文化——数学类型在设计时就考虑了如何被远程监控。

6. **BlendTransform 的工程权衡**：不直接对矩阵插值，而是分解为 quat+scale+translate，是骨骼动画的标准做法，保证插值结果仍是合法的刚体变换。

7. **新旧噪声并存**：`PerlinNoise`（经典）与 `SimplexNoise`（改进）并存，新代码倾向 Simplex，旧资源仍用 Perlin，二者接口风格不同（PerlinNoise 是松散函数集合，SimplexNoise 是带八度配置的类）。
