---
title: IL(lskmc) 十月开发内容
published: 2025-10-31
description: "星器内部资料"
tags: ["dev","lskmc(IL)","C#","WPF"]
category: 项目规划
draft: false
author: "黄子郬"
pinned: false
---
# fabric安装 -lskmcL启动内核
fabric - 原版 json 合并
```csharp
//downloadL.cs
public static void MergeFabricAndVanillaJson(string fabricJsonPath, string vanillaJsonPath, string outputJsonPath,string FabricVersion)
{
    var fabricNode = JsonNode.Parse(File.ReadAllText(fabricJsonPath));
    var vanillaNode = JsonNode.Parse(File.ReadAllText(vanillaJsonPath));

    // mainClass 用 fabric 的 client 字段
    string mainClass = fabricNode?["mainClass"]?["client"]?.ToString() ?? vanillaNode?["mainClass"]?.ToString();
    vanillaNode["mainClass"] = mainClass;

    // 合并 libraries.common 到原版 libraries
    var mergedLibs = new JsonArray();

    // fabric libraries.common
    var fabricLibs = fabricNode?["libraries"]?["common"]?.AsArray();
            if (fabricLibs != null)
            {
                foreach (var lib in fabricLibs)
                    mergedLibs.Add(lib.DeepClone());
                // 添加 Fabric Loader 本体
                mergedLibs.Add(new JsonObject
                {
                    ["name"] = $"net.fabricmc:fabric-loader:{FabricVersion}",
                    ["url"] = "https://maven.fabricmc.net/",
                });
            }
            else
            {
                Console.WriteLine($"Warning: 'libraries.common' not found in {fabricJsonPath}");
            }
    // 原版 libraries
    var vanillaLibs = vanillaNode?["libraries"]?.AsArray();
    if (vanillaLibs != null)
    {
        foreach (var lib in vanillaLibs)
            mergedLibs.Add(lib.DeepClone());
    }

    vanillaNode["libraries"] = mergedLibs;

    // 保存到文件
    File.WriteAllText(outputJsonPath, vanillaNode.ToJsonString(new JsonSerializerOptions { WriteIndented = true }));
}
```
·如何使用？
```csharp
 DownLoadL.MergeFabricAndVanillaJson("fabric-loader-0.17.2.json", "1.21.8.json", "versions\\1.21.8-fabric0.17.2\\1.21.8-fabric0.17.2.json","0.17.2");
```

- - -
# fabric安装 - IL UI实现