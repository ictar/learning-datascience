原文：[How I Used Python and Open Data to Build an Interactive Map of Edinburgh’s Beergardens](https://walkenho.github.io/beergarden-happiness-with-python/)

---

又名：地理编码桌椅许可，以享受室外冷饮。


随着夏天最终抵达，我想知道，在我的家乡爱丁堡，哪里是享用一杯美味的冷饮（含酒精或不含酒精）的好去处。因此，我将关于椅子许可的开放数据集和一些地理编码相结合，创建了一个爱丁堡外部座位的交互式地图。

## 背景和项目描述

在过去的几年里，英国政府一直致力于[开放数据](https://www.computerworlduk.com/data/how-uk-government-uses-open-data-3683332/)，而爱丁堡市议会也不例外。在 <https://edinburghopendata.info> 上面，你可以找到一个数据集列表，包含有关公共生活的许多方面的信息（虽然某些文件肯定需要进行更新了）。例如[此](https://data.edinburghopendata.info/dataset/tables-and-chairs-permits)页有一个文件，其中包含有关 2014 年桌椅许可证的详细信息。幸运的是，可以[在这里](http://www.edinburgh.gov.uk/download/downloads/id/11854/tables_and_chairs_permits.csv)找到最新版本。注意，虽然两个文件的文件结构相同，但是标题不同。因此，如果想要查看历史数据，你需要相应调整下面的代码。该文件包含有权放置椅子的场所名称和地址，以及一些其他信息。该项目以这个文件为基础，分成四个部分：

  * 获取并加载许可文件
  * 使用开放街道地图 API，获取每个处所以及场所类别的经纬度
  * 清理和分类场所类别
  * 使用 folium，在地图上绘制场所



话不多说，让我们开始吧。可以在我的 [GitHub](https://github.com/walkenho/tales-of-1001-data/blob/master/beergarden_happiness_with_python/beergarden_happiness_with_python.ipynb) 上找到完整的 notebook。

## 第 0 步：设置

首先，导入库。

```py
    import pandas as pd
    import requests
    import wget
    
    import folium
    from folium.plugins import MarkerCluster
    
```

## 第 1 步：获取数据

使用 wget 下载文件，并将其读入 pandas 数据帧中。确保设置编码，因为该文件含有特殊字符（列表中有许多咖啡馆）。
k
```py
    filename = wget.download("http://www.edinburgh.gov.uk/download/downloads/id/11854/tables_and_chairs_permits.csv")
    
    df0 = pd.read_csv(filename, encoding = "ISO-8859-1")
    df0.head()
    
```

| Permit | Premises Name | Premises Address | Start date | End date | Ext | Table Area  
---|---|---|---|---|---|---|---  
0 | 5.0 | The Standing Order | 62-66 George Street | 01/04/2019 | 31/03/2020 | 22:00 | 6.0m x 1.5m &amp; 6.0m x 1.2m  
1 | 6.0 | Bushwick Bar and Grill | 7-11 East London Street | 01/06/2019 | 01/09/2019 | 22:00 | 7.85m x 1.8m and 2.5m x 1.8m  
2 | 7.0 | La Barantine | 202 Bruntsfield Place | 01/01/2019 | 31/12/2019 | NaN | 2.0m x 1.2m  
3 | 10.0 | The Inverleith | 10 Bowhill Terrace | 18/04/2019 | 31/08/2019 | NaN | 4.1m x 3.1m  
4 | 12.0 | Angels Share Hotel | 7-11 Hope Street | 01/05/2019 | 30/04/2020 | 22:00 | 4.8m x 1.4m and 1.5m x 1.4m and 1.5m x 1.4m  
  
快速浏览数据，会发现其中有些重复数据。这主要是因为具有不同的开始和结束时间的多个许可。一个清理好方法是过滤日期，但老实说，现在我不那么在乎。所以，我只保留了场所名称和地址，并且删除重复项。（注意：该文件还包含了表区域信息，将来我有可能会重新访问这些信息）。删除重复项后，剩下 389 行数据，包含场所名称和地址。

```py
    # dropping duplicate entries
    df1 = df0.loc[:, ['Premises Name', 'Premises Address']]
    df1 = df1.drop_duplicates()
    
    # in 2012: 280
    print(df1.shape[0])
    
```

```
    389    
```

旁注：在 2014 年夏天，只有 280 个场所有桌椅许可。露天文化确实火起来了，这就是证明 :)

## 第 2 步：获取每个场所的经纬度

如果我们想要在地图上看到这些场所，那么只有地址是不够的，需要 GPS 坐标。有不同的 API 允许你查询一个地址的经纬度（这个过程称为 [geocoding](https://en.wikipedia.org/wiki/Geocoding)）。一个选择是使用 [Google Maps API](https://developers.google.com/maps/documentation/)，但它有警告。[OpenStreetMap](https://www.programmableweb.com/api/openstreetmap) API 提供了相同的功能，并且是免费使用的，而且对我而言，其返回的结果够用了。

我们使用 pandas 的 map 函数来获取每一行的请求响应。在查询 API 后，删除所有没有响应的行。再说一次，对于我所损失的少数场所数据（大概 20 个），我并不感到太多的烦恼。剩下的数据还有很多。

```py
    def query_address(address):
        """Return response from open streetmap.
        
        Parameter:
        address - address of establishment
        
        Returns:
        result - json, response from open street map
        """
        
        url = "https://nominatim.openstreetmap.org/search"
        parameters = {'q':'{}, Edinburgh'.format(address), 'format':'json'}
        
        response = requests.get(url, params=parameters)
        # don't want to raise an error to not stop the processing
        # print address instead for future inspection
        if response.status_code != 200:
            print("Error querying {}".format(address))
            result = {}
        else:
            result = response.json()
        return result
    
```

```py
    df1['json'] = df1['Premises Address'].map(lambda x: query_address(x))
    
    # drop empty responses
    df2 = df1[df1['json'].map(lambda d: len(d)) > 0].copy()
    print(df2.shape[0])
    
```

```
    374  
```

看看响应中的 json 字段，我们发现，除了坐标外，该 API 还返回一个名为“type”的字段，该字段包含此地址的场所类型。我将该信息与坐标一起添加到了数据帧中。

```py
    # extract relevant fields from API response (json format)
    df2['lat'] = df2['json'].map(lambda x: x[0]['lat'])
    df2['lon'] = df2['json'].map(lambda x: x[0]['lon'])
    df2['type'] = df2['json'].map(lambda x: x[0]['type'])
    
```

最常见的场所类型是咖啡馆、酒馆、餐馆、第三产业（tertiary）和房子：
```py
    df2.type.value_counts()[:5]
```

```
    cafe          84
    pub           69
    restaurant    66
    tertiary      33
    house         27
    Name: type, dtype: int64
    
```

### 第 3 步：分配场所类别

我最感兴趣的是区分两种场所类型：那些出售咖啡并且更有可能在白天开放的场所（如咖啡馆和面包店），以及出售啤酒并更有可能在晚上开放的场所（像小酒馆和餐馆）。因此，我想要将场所分为三类：

  * 第 1 类：日间活动场所（咖啡店、面包店、熟食店，冰淇淋店）
  * 第 2 类：小酒馆、餐馆、快餐店和酒吧
  * 第 3 类：其他



为此，我有两个信息来源：场所名称和 OpenStreetMap 返回的类型。看看数据，我们发现类型是第一个良好的指标但也有许多地方标记错误，或者根本没有标记。因此，我采用两步法：1）根据 OpenStreetMap 类型分配类别 2）使用其名称清洗数据，此步骤将会覆盖步骤 1）。为了清洗数据，如果场所名称包含某些关键元素（例如，“cafte”、“coffee”等咖啡馆类似因素，以及“restaurant”、“inn”等餐馆和小酒馆类似因素），就确定推翻 OpenStreetMap 分类。这会有些错分，比如把 Cafe Andaluz 分类为咖啡店，但是在大多数情况下工作良好。特别是，它似乎最符合咖啡店分类模式，而咖啡店可能会在白天开门。因此，这对我而言适用。当然，如果少于 400 项，那么可以人工浏览列表，并为每一项分配正确的类别。但是，我对创建一个可以很容易转移到其他地方使用的过程比较感兴趣。因此，专门针对爱丁堡的人工干预并不适用。

#### 第 3a 步：根据 OpenStreetMap 类型分配场所类别

```py
    def define_category(mytype):
        if mytype in ['cafe', 'bakery', 'deli', 'ice_cream']:
            category = 1
        elif mytype in ['restaurant', 'pub', 'bar', 'fast_food']:
            category = 2
        else:
            category = 3
        return category
    
    # assign category according to OpenStreetMap type
    df2['category'] = df2['type'].map(lambda mytype: define_category(mytype))
    
```

#### 第 3b 步：根据场所名称覆盖类别

```py
    def flag_premise(premisename, category):
        """Flag premise according to its name.
        
        Parameter: 
        premisename - str
        
        Returns:
        ans - boolean
        """
        prem = str(premisename).lower()
        if ((category == 'coffeeshop' and ('caf' in prem 
                                           or 'coffee' in prem 
                                           or 'Tea' in str(premisename) 
                                           or 'bake' in prem 
                                           or 'bagel' in prem 
                                           or 'roast' in prem))
             or 
            (category == 'restaurant' and ('restaurant' in prem 
                                           or 'bar ' in prem 
                                           or 'tavern' in prem 
                                           or 'cask' in prem 
                                           or 'pizza' in prem
                                           or 'whisky' in prem
                                           or 'kitchen' in prem
                                           or 'Arms' in str(premisename)
                                           or 'Inn' in str(premisename) 
                                           or 'Bar' in str(premisename)))):
            ans = True
        else:
            ans = False
        return ans
    
    # flag coffee shops and restaurants according to their names
    df2['is_coffeeshop'] = df2['Premises Name'].map(lambda x: flag_premise(x, category='coffeeshop'))
    df2['is_restaurant'] = df2['Premises Name'].map(lambda x: flag_premise(x, category='restaurant'))
    
```

快速检查显示，重新调整似乎是合理的：

```py
    # show some differences between classification by name and by type returned by the API
    df2.loc[(df2.is_coffeeshop) & (df2.type != 'cafe'), ['Premises Name', 'type']].head(10)
    
```

| Premises Name | type  
---|---|---  
19 | One20 Wine Café | bar  
27 | Café Habana | theatre  
38 | Southern Cross Café | fast_food  
73 | Café Rouge | restaurant  
95 | Café Andaluz | restaurant  
114 | The Manna House Bakery | tertiary  
115 | Crumbs Café | house  
146 | Cafe Renroc | residential  
185 | Hard Rock Café | tertiary  
227 | Snax Café | house  
  
我为标记为餐厅或者咖啡店的场所重新分配了类别。如果一个场景被同时标记为这两种，那么则优先使用咖啡店类别：

```py
    # re-set category if flagged as restaurant or coffeeshop through name
    df2.loc[df2.is_restaurant, 'category'] = 2
    df2.loc[df2.is_coffeeshop, 'category'] = 1
    
```

## 第 4 步：可视化

最后，我们使用 Python 的 Folium 包来将结果可视化为地图上的标记。将单个点添加到 `MarkerClusters` 允许我们在同一区域有太多符号的情况下，将这些符号汇总成组。为每个类别创建单独的集群让我们可以使用 `LayerControl` 选项，单独切换每个类别。我们使用“fa”前缀来使用 font-awesome（而不是 standard glyphicon）符号。

```py
    # central coordinates of Edinburgh
    EDI_COORDINATES = (55.953251, -3.188267)
      
    # create empty map zoomed in on Edinburgh
    map = folium.Map(location=EDI_COORDINATES, zoom_start=12)
    
    # add one markercluster per type to allow for individual toggling
    coffeeshops = MarkerCluster(name='coffee shops').add_to(map)
    restaurants = MarkerCluster(name='pubs and restaurants').add_to(map)
    other = MarkerCluster(name='other').add_to(map)
    
    # add coffeeshops to the map
    for chairs in df2[df2.category == 1].iterrows():
        folium.Marker(location=[float(chairs[1]['lat']), float(chairs[1]['lon'])], 
                      popup=chairs[1]['Premises Name'],
                     icon=folium.Icon(color='green', icon_color='white', icon='coffee', angle=0, prefix='fa'))\
        .add_to(coffeeshops)
        
    # add pubs and restaurants to the map
    for chairs in df2[df2.category == 2].iterrows():
        folium.Marker(location=[float(chairs[1]['lat']), float(chairs[1]['lon'])], 
                      popup=chairs[1]['Premises Name'],
                     icon=folium.Icon(color='blue', icon='glass', prefix='fa'))\
        .add_to(restaurants)
        
    # add other to the map
    for chairs in df2[df2.category == 3].iterrows():
        folium.Marker(location=[float(chairs[1]['lat']), float(chairs[1]['lon'])], 
                      popup=chairs[1]['Premises Name'],
                     icon=folium.Icon(color='gray', icon='question', prefix='fa'))\
        .add_to(other)
        
    # enable toggling of data points
    folium.LayerControl().add_to(map)    
        
    display(map)
    
```
> 译注：[原文](https://walkenho.github.io/beergarden-happiness-with-python/)有结果图

## 补充第 5 步：将地图保存为 png

我希望有该地图的一份屏幕截图，以便能够将地图的静态版本嵌入到我的 Medium 文章（不接受动态版本）中。我发现获得静态版本（不仅仅是截屏）的最佳方式是，以 HTML 格式保存地图，然后使用 Selenium 来保存该 HTML 的屏幕截图。这就是实现方式（归于[这篇 stackoverflow 文章](https://stackoverflow.com/questions/40208051/selenium-using-python-geckodriver-executable-needs-to-be-in-path)的 Selenium 部分）。

注意：为了能够正常使用以下代码，你需要安装 geckodriver。. Download the file from [在这里](https://github.com/mozilla/geckodriver/releases) 下载该文件，然后将其放到 /usr/bin/local（对于 Linux 机器）。

```py
    import os
    import time
    from selenium import webdriver
    
    # save map
    fn = 'beergarden_happiness_map.html'
    tmpurl = 'file:///{path}/{mapfile}'.format(path=os.getcwd(),mapfile=fn)
    map.save(fn)
    
    # download screenshot of map
    delay = 5
    browser = webdriver.Firefox()
    browser.get(tmpurl)
    # give the map tiles some time to load
    time.sleep(delay)
    browser.save_screenshot('{mapname}.png'.format(mapname=fn.split('.')[0]))
    browser.quit()
    
```

## 概要

在这篇文章中，我们从爱丁堡议会下载了一份包含桌椅许可的开放数据集。然后基于许可地址，使用 Open Street Map API 来获取许可的类型和 GPS 位置。在根据场所名称进行一些额外的数据清理后，我们将场所分为三种类别：“咖啡店”、“酒吧/餐厅”和“其他”。然后将其绘制在交互式地图上。接着以 HTML 的格式保存该地图并随后转换为 png 格式。

## 结论

现在，我们拥有了一份有效的爱丁堡露天啤酒店和咖啡店地图，可以坐在外面，享用美味的冰咖啡或者冰镇啤酒，以享受夏天。祝健康！:)

![](https://walkenho.github.io/images/beergarden-happiness-drink.jpg)
↑ 地图很好用：阳光下的工作后冷饮 :)