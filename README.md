
天气雷达的基本要素有很多,特别是双偏振雷达更多,但业务场景经常使用的一般为基本反射率,基本速度这两种要素


接下来我们以基本反射率为例,其他的要素也是一样的,一通百通


首先我们做基本反射率的图需要确定做哪一个仰角层,因为雷达体扫模式的扫描是不同仰角进行扫描的,常规的雷达一般是9个仰角


![](https://img2024.cnblogs.com/blog/1603698/202412/1603698-20241217161403488-833996800.png)


按照上图就很明显的知道体扫模式的扫描是怎样的情况了


由于雷达基本的要素都是径向数据,我们先以图的方式看径向数据如何绘制


![](https://img2024.cnblogs.com/blog/1603698/202412/1603698-20241217163924363-1217037355.png)


 可以看到,要素产品是由n条径向数据组成,所有的径向数据从O点根据方位角以及距离,正好形成一个完美的圆


所以我们画一层仰角的图就需要距离,方位角,数值即可


现在我们就画基本反射率第一层的径向数据图,也就是0\.5度仰角




```
MeteoDataInfo meteoDataInfo = new MeteoDataInfo();
meteoDataInfo.openData("D:\\tls\\Z_RADR_I_站点_20220407233130_O_DOR_CC_CAP_FMT.bin");
CMARadarBaseDataInfo info = (CMARadarBaseDataInfo) meteoDataInfo.getDataInfo();
//色阶颜色
int[][] cols = {
        {255, 255, 255},
        {102, 255, 255},
        {102, 255, 255},
        {0, 162, 232},
        {86, 225, 250},
        {3, 207, 14},
        {26, 152, 7},
        {255, 242, 0},
        {217, 172, 113},
        {255, 147, 74},
        {255, 0, 0},
        {204, 0, 0},
        {155, 0, 0},
        {236, 21, 236},
        {130, 11, 130},
        {184, 108, 208}
};
//色阶数值
double[] levs = new double[]{Integer.MIN_VALUE,0, 5, 10, 15, 20, 25, 30, 35, 40, 45, 50, 55, 60, 65, 70,Integer.MAX_VALUE};
//根据RGB返回Color对象
Color[] colors = ColorUtil.getColorsFromStyle(cols);
//根据数值和颜色生成绘图所用的色阶
LegendScheme ls = LegendManage.createGraduatedLegendScheme(levs,colors, ShapeTypes.POLYGON);
List list = new ArrayList<>();
//提前准备好所需要的要素
//反射率
list.add("dBZ");
//方位角
list.add("azimuthR");
//距离
list.add("distanceR");
//仰角
list.add("elevationR");
//从基数据中取出对应数据绘制成图层
VectorLayer layer = MeteoinfoUtil.getRadarArray(info,list,0,ls);
MapView view = new MapView();
view.addLayer(layer);
//导出图片
MeteoinfoUtil.export(view,1200,"D:/tls/r.png");
```



```
ColorUtil.getColorsFromStyle这个方法很简单,我随便封装了一下,只是为了满足根据RGB返回Color对象,所以代码就不公布了而MeteoinfoUtil.getRadarArray是取值的核心,我们来具体讲解一下
```



```
public static VectorLayer getRadarArray(CMARadarBaseDataInfo info, List pros, int i, LegendScheme ls) throws Exception {
        //站点纬度
        Attribute lat = info.findGlobalAttribute("StationLatitude");
        //站点经度 
        Attribute lon = info.findGlobalAttribute("StationLongitude");
        //海拔高度 
        Attribute high = info.findGlobalAttribute("AntennaHeight");
        List names = info.getVariableNames();
        if (!names.containsAll(pros)) {
            return null;
        }
        //从基数据中读取所需要素的径向数据
        Array ay = info.read(pros.get(0));
        Array[] arrays = new Array[4];
        //获取径向数据的条数,也就是方位角个数
        int azimuthNum = ay.getShape()[1];
        //获取一条径向数据的数据块个数
        int dataBlockNum = ay.getShape()[2];
        //---------获取i层的径向数据,格式布局为Z*Y*X     
        //设定读取数据的起始位置
        int[] origin = new int[]{i, 0, 0};
        //设定所读取数据数量
        int[] size = new int[]{1, azimuthNum, dataBlockNum};
        //设定读取数据的步幅 
        int[] stride = new int[]{1, 1, 1};
        //取出要素的单层径向数据
        arrays[0] = info.read(pros.get(0), origin, size, stride);
        //---------获取i层的方位角
        origin = new int[]{i, 0};
        size = new int[]{1, azimuthNum};
        stride = new int[]{1, 1};
        //取出单层的方位角
        arrays[1] = info.read(pros.get(1), origin, size, stride);
        //-----------获取i层数据块的距离
        origin = new int[]{0};
        size = new int[]{dataBlockNum};
        stride = new int[]{1};
        arrays[2] = info.read(pros.get(2), origin, size, stride);
        //------------获取i层的仰角数据
        origin = new int[]{i, 0};
        size = new int[]{1, azimuthNum};
        stride = new int[]{1, 1};
        arrays[3] = info.read(pros.get(3), origin, size, stride);
        //------------获取所需数据end--------------
        //将方位角和仰角从角度转化为弧度
        Array azi = ArrayMath.toRadians(arrays[1]);
        Array ele = ArrayMath.toRadians(arrays[3]);
        //使用距离和方位角(弧度)创建二维矩阵,这一步更重要的是为了适用Transform.antennaToCartesian
        Array[] a = ArrayUtil.meshgrid(arrays[2], azi);
        Array dis = a[0];
        azi = a[1];
        List list = new ArrayList();
        list.add(dis.getShape()[1]);
        重组对象格式
        ele = ele.reshape(new int[]{azimuthNum, 1});
        //塞数据进去
        ele = ArrayUtil.repeat(ele, list, 1);
        int h = Integer.parseInt(high.getValue().toString().trim());
        //天线坐标转换为笛卡尔坐标
        Array[] aa = Transform.antennaToCartesian(dis, azi, ele, h);
        String projection = String.format("+proj=aeqd  +lon_0=%s  +lat_0=%s",
                Double.parseDouble(lon.getValue().toString().trim()),
                Double.parseDouble(lat.getValue().toString().trim()));
        //确定投影,方便后续加地图或地理信息
        ProjectionInfo projectionInfo = factory(new CRSFactory().createFromParameters("custom",
                projection));
        //坐标重投影 
        Array[] xy = Reproject.reproject(aa[0], aa[1], projectionInfo, LONG_LAT);
        //绘制图层
        VectorLayer layer = DrawMeteoData.meshLayer(xy[0], xy[1], arrays[0], ls);
        return layer;
    }
```


 


这种方法是最复杂也是速度较慢的,但同时也是不会丢任何数据的方法,下一节我们换种使用了插值法的绘制方法,使用了插值后,性能以及速度就得到了大大的提升　　


 本博客参考[樱花宇宙官网](https://yzygzn.com)。转载请注明出处！
