# `run_d03_to_d04_ndown` 源码改动说明

## 范围

这份说明只归纳为 `run_d03_to_d04_ndown` 这条流程做过的源码改动，重点是：

1. `ndown` 输出阶段的城市参数传播逻辑
2. `Registry` 中城市字段的注册/插值/拷贝规则
3. `phys` 中为城市致命错误增加的诊断输出

仓库当前 `main` 分支对应的相关提交为：

- `14c5f21` `Add LCZ urban field propagation fixes for real and ndown`
- `1f13fe3` `Add runtime diagnostics for urban fatal threshold`
- `d8df546` `Add start_em dz8w diagnostics for urban fatal cell`
- `1b0729d` `Track live WRF diagnostics sources for runtime debugging`

## 改了什么

### 1. `Registry/Registry.EM_COMMON`

位置：

- `Registry/Registry.EM_COMMON:833-839`
- `Registry/Registry.EM_COMMON:947`

改动内容：

- 给 `LP_URB2D`、`HI_URB2D`、`LB_URB2D`、`HGT_URB2D`、`MH_URB2D`、`STDH_URB2D`、`LF_URB2D` 增加了 `i012rd=(interp_mask_land_field:lu_index)u=(copy_fcnm)`。
- 给 `UTYPE_URB2D` 增加了 `i012rd=(interp_fcni)u=(copy_fcni)`。

实际意义：

- 这些字段不再只是“存在于源文件里”，而是被明确告诉 WRF 在 `real/ndown` 过程中如何插值、如何复制。
- 这是城市参数能否从上游输入传播到嵌套输出里的关键一步。

### 2. `dyn_em/module_initialize_real.F`

位置：

- `dyn_em/module_initialize_real.F:3065-3073`

改动内容：

- 在 `use_wudapt_lcz = 1` 时，对 `FRC_URB2D > 0` 且 `LU_INDEX` 在 `31-41` 的格点：
  - `ivgtyp = NINT(lu_index)`
  - `vegcat = ivgtyp`
  - `UTYPE_URB2D = ivgtyp - 30`

实际意义：

- `real.exe` 初始化后，LCZ 城市类型不会只停留在 `LU_INDEX`，而是同步到后续城市物理真正会读到的 `IVGTYP/VEGCAT/UTYPE_URB2D`。

### 3. `main/ndown_em.F`

位置：

- `main/ndown_em.F:673-680`

改动内容：

- 在 `ndown` 生成 `nested_grid` 时，增加了与 `real.exe` 同样的一段 `use_wudapt_lcz` 逻辑：
  - 对城市格点回写 `ivgtyp`
  - 同步 `vegcat`
  - 重建 `UTYPE_URB2D`

实际意义：

- 这部分直接改的是`ndown` 输出逻辑。
- 即使上游 `LU_INDEX/FRC_URB2D` 已经是对的，`ndown` 以前仍可能把城市类型链条断在 `IVGTYP/UTYPE_URB2D` 上；现在会在输出前重新补齐。

### 4. `phys/module_sf_urban.F`

位置：

- `phys/module_sf_urban.F:787-793`

改动内容：

- 在原来的 fatal
  - `ZDC + Z0C + 2m is larger than the 1st WRF level ...`
  之前，增加了三组输出：
  - `URBAN FATAL DIAG UTYPE,ZA,ZR,ZDC,Z0C = ...`
  - `URBAN FATAL DIAG MH,STDH,LP,HGT,FRC,LB = ...`
  - `URBAN FATAL DIAG LF(1:4) = ...`

实际意义：

- fatal 前会把冠层高度、粗糙度相关量、建筑统计量一起打出来，能直接判断是 `ZA` 太低，还是 `MH/HGT/LP/LB/LF` 本身异常。

### 5. 额外加入的运行时诊断

位置：

- `dyn_em/start_em.F:2107-2110`
- `phys/module_sf_noahdrv.F:807`
- `phys/module_sf_noahdrv.F:1401-1429`
- `phys/module_sf_noahdrv.F:4702-4730`

改动内容：

- 在固定格点 `(i,j)=(212,227)` 增加了：
  - `DZ8W` / `ZLVL` 诊断
  - 调用 `urban()` 前后的城市参数诊断

实际意义：

- 这部分不是改物理结果本身，而是为了追 `urban fatal` 时，确认第一层厚度、进入 `urban()` 前的参数、以及 `urban()` 返回后的量到底是什么。

## 前后输出有什么不同

### 一、`wrfinput` 城市字段是否真正被带进去

对比文件：

- 改前备份：`run_d03_to_d04_ndown/wrfinput_d01.before_backfill_urban_fields_20260509`
- 改后产物：`run_d03_to_d04_ndown/wrfinput_d01_ndown`

两者共同点：

- `FRC_URB2D > 0` 的城市格点数都为 `57372`

改前：

- `MH_URB2D` 在全部 `57372` 个城市格点上都是 `0`
- `STDH_URB2D` 在全部 `57372` 个城市格点上都是 `0`
- `LP_URB2D` 缺失
- `LB_URB2D` 缺失
- `HGT_URB2D` 缺失
- `LF_URB2D` 虽然变量存在，但城市格点上全为 `0`

改后：

- `MH_URB2D` 在 `57372` 个城市格点上全部变为非零，最大值 `50.0`
- `STDH_URB2D` 在 `57372` 个城市格点上全部变为非零，最大值 `12.5`
- `LP_URB2D` 不再缺失，`57372` 个城市格点全部非零，最大值 `0.6470588445663452`
- `LB_URB2D` 不再缺失，`57372` 个城市格点全部非零，最大值 `2.8947367668151855`
- `HGT_URB2D` 不再缺失，`57372` 个城市格点全部非零，最大值 `50.000003814697266`
- `LF_URB2D` 在四个方向总计 `229488` 个城市分量上全部非零，最大值 `2.3684210777282715`

结论：

- 改前是“有城市掩膜，但关键城市形态参数没有跟着进去”。
- 改后变成“城市掩膜和城市形态参数一起进入了 `ndown`/城市物理链条”。

### 二、`ndown` 输出文件体量明显变大

从作业日志可见：

- `slurm-ndown-2496.out:17-18`
  - `wrfbdy_d02 = 3.2G`
  - `wrfinput_d02 = 38M`
- `slurm-ndown-2597.out:17-18`
  - `wrfbdy_d02 = 9.3G`
  - `wrfinput_d02 = 150M`
- `slurm-ndown-2604.out:17-18`
  - `wrfbdy_d02 = 9.3G`
  - `wrfinput_d02 = 152M`

结论：

- 相比 4 月 29 日那批早期 `ndown` 输出，5 月 9 日源码修补后的 `ndown` 产物明显更大。
- 这与更多城市字段被注册并真正进入输出文件是一致的。

### 三、fatal 日志从“只报错”变成“报错前先吐诊断”

改前实际输出只有 fatal 主句：

```text
ZDC + Z0C + 2m is larger than the 1st WRF level - Stop in subroutine urban - change ZDC and Z0C
```

改后源码会在 fatal 前多输出三组诊断：

```text
URBAN FATAL DIAG UTYPE,ZA,ZR,ZDC,Z0C = ...
URBAN FATAL DIAG MH,STDH,LP,HGT,FRC,LB = ...
URBAN FATAL DIAG LF(1:4) = ...
ZDC + Z0C + 2m is larger than the 1st WRF level - Stop in subroutine urban - change ZDC and Z0C
```

说明：

- 当前目录里没有保留那次 fatal 的旧 `rsl` 副本，所以这一节的“输出前后差异”依据是源码中新增的打印语句本身。
- 这部分结论是确定的，因为新增语句就在 `phys/module_sf_urban.F:787-793`。

## 总结

这批改动本质上做了两件事：

1. 把 LCZ/城市形态参数真正接入 `real.exe -> ndown.exe -> wrf.exe` 这一条链路，避免城市格点只有掩膜、没有完整形态参数。
2. 给城市 fatal 增加足够细的运行时诊断，让后续排错不再只看到一句泛化报错。

如果只看 `run_d03_to_d04_ndown` 这条流程，最关键的源码修改是：

- `Registry/Registry.EM_COMMON`
- `dyn_em/module_initialize_real.F`
- `main/ndown_em.F`

最关键的输出变化是：

- 城市参数从“缺失/全零”变成“在全部城市格点上被正确写入”
- `ndown` 产物明显变大
- `urban fatal` 前开始带出可直接定位问题的诊断信息
