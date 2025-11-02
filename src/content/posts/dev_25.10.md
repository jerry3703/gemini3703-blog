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
## fabric - 原版 json 合并
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
## ·如何使用？
```csharp
 DownLoadL.MergeFabricAndVanillaJson("fabric-loader-0.17.2.json", "1.21.8.json", "versions\\1.21.8-fabric0.17.2\\1.21.8-fabric0.17.2.json","0.17.2");
```

- - -
# fabric安装 - IL UI实现
### 前端
```xaml
<ui:Page x:Class="lskmc2f.Views.Pages.DownLoadGameDetail"
      xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
      xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
      xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006" 
      xmlns:d="http://schemas.microsoft.com/expression/blend/2008" 
      xmlns:local="clr-namespace:lskmc2f.Views.Pages"
      xmlns:ui="http://schemas.inkore.net/lib/ui/wpf/modern"
      mc:Ignorable="d" 
      d:DesignHeight="610" d:DesignWidth="450"
      Title="DownLoadGameDetail">

    <Grid Background="White">
        <TextBlock HorizontalAlignment="Left" Height="30" Margin="10,10,0,0" TextWrapping="Wrap" VerticalAlignment="Top" Width="415" FontSize="20"><Run Language="zh-cn" Text="{Binding DisplayGame}"/></TextBlock>
        <ScrollViewer VerticalScrollBarVisibility="Auto" Margin="0,63,0,0" >
            <Grid>
                <ItemsControl ItemsSource="{Binding ModLoader}" Margin="0,0,0,55">
                    <ItemsControl.ItemsPanel>
                        <ItemsPanelTemplate>
                            <StackPanel HorizontalAlignment="Stretch"/>
                        </ItemsPanelTemplate>
                    </ItemsControl.ItemsPanel>
                    <ItemsControl.ItemTemplate>
                        <DataTemplate>
                            <Expander Style="{StaticResource MicaExpanderStyle}" Margin="5" ExpandDirection="Down"  Header="{Binding}" >
                                <Expander.HeaderTemplate>
                                    <DataTemplate>
                                        <StackPanel Orientation="Horizontal" VerticalAlignment="Center" >
                                            <!--  <Image Source="{Binding Icon}" Width="24" Height="24" Margin="0,0,8,0"/> -->
                                            <TextBlock Text="{Binding ModLoaderType}" HorizontalAlignment="Left" FontWeight="Bold" FontSize="16" Foreground="Black"/>
                                        </StackPanel>
                                    </DataTemplate>
                                </Expander.HeaderTemplate>
                                <ItemsControl ItemsSource="{Binding SubVersions}">
                                    <ItemsControl.ItemTemplate>
                                        <DataTemplate>
                                            <Button Content="{Binding Version}" Margin="2"
                                                        Command="{Binding ChooseCommand}"
                                                        CommandParameter="{Binding}"
                                                         HorizontalAlignment="Stretch"
                                                         HorizontalContentAlignment="Left" 
                                                        />
                                        </DataTemplate>
                                    </ItemsControl.ItemTemplate>
                                </ItemsControl>
                            </Expander>
                        </DataTemplate>
                    </ItemsControl.ItemTemplate>
                </ItemsControl>
            </Grid>
        </ScrollViewer>
        <Button Content="开始下载" Margin="0,554,0,0" VerticalAlignment="Top" Height="46" Width="430" HorizontalAlignment="Center" Command="{Binding StartDownloadCommand}" />
    </Grid>

</ui:Page>

```
### 后端
```csharp
//DownLoadGameDetailViewModel.cs
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;
using CommunityToolkit.Mvvm.Messaging;
using lskmc2f.Views.Pages;
using System;
using System.Collections.Generic;
using System.Collections.ObjectModel;
using System.Diagnostics;
using System.IO;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows;
using CommunityToolkit.Mvvm;
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Messaging;
using lskmc2f.Messages;
namespace lskmc2f.ViewModels.Pages
{
    //控件动态生成之定义
    public partial class ModLoaderViewModel : ObservableObject
    {
        [ObservableProperty]
        public required string modLoaderType ;

        [ObservableProperty]
        public ObservableCollection<FabricLoaderVersionViewModel> subVersions = new();//模组加载器版本嵌套在这里再传出


    }


    public partial class FabricLoaderVersionViewModel : ObservableObject
    {
        [ObservableProperty]
        public string version;
        [ObservableProperty]
        public string minecraftVersion = "1.21.8";
        [ObservableProperty]
        public string displayname;
        public string url;
        [RelayCommand]
        public  Task Choose()
        {
            DownLoadGameDetailViewModel.choose_mod_type = "Fabric";
        DownLoadGameDetailViewModel.choose_fabric_version = this.version;
          //  gameName = "Fabric";
            //MessageBox.Show("xuanz" + Fabric.version);
            return Task.CompletedTask;
        }
    }
    public partial class VanillaVersionViewModel : ObservableObject
    {
        [ObservableProperty]
        public string version;
        [ObservableProperty]
        public string url ;
    }
    public partial class DownLoadGameDetailViewModel : ObservableObject
    {
        public static string choose_mod_type = "Vanilla";
        public static string choose_fabric_version ="默认版本";
        public static string VanillaUrl;
        [ObservableProperty]
        public static string gameName = "1.21.8";
        [ObservableProperty]
        public static string displayGame = "默认下载";
        [ObservableProperty]
        private ObservableCollection<ModLoaderViewModel> modLoader = new ();
        private static string MinecraftVersion = "默认版本";

        [RelayCommand]
        public static Task StartDownload()
        {
            if (choose_mod_type == "Fabric")
            {
                MessageBox.Show(choose_mod_type+MinecraftVersion);
                gameName+=$"Fabric{choose_fabric_version}";
                var subversion = new FabricLoaderVersionViewModel() { version = choose_fabric_version,MinecraftVersion = MinecraftVersion ,displayname = gameName ,url = VanillaUrl };
                WeakReferenceMessenger.Default.Send(new lskmc2f.Messages.FabricLoaderDownloadMessage(subversion));
            }
            else
            {
                MessageBox.Show("Forge下载功能暂未开放！");
            }
            return Task.CompletedTask;
        }
        
        public DownLoadGameDetailViewModel()
        { // 注册消息
             WeakReferenceMessenger.Default.Register<Messages.DownloadManagerMessage>(this, async (r, m) =>
         {//下载版本侧边展开
             gameName = m.Value.id;
             displayGame = " 下载"+m.Value.id;
             OnPropertyChanged(nameof(gameName));
             MinecraftVersion = m.Value.id;
             VanillaUrl = m.Value.Url;
            });
            //获取json
            //解析json
            
            Debug.WriteLine("开始fabric加载版本信息...");
            string jsonPath = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.Desktop), "loader");
            // 读取 JSON 文件
            try
            {
                var json = System.IO.File.ReadAllText(jsonPath); // 修正：同步读取文件内容
                var Fabricroot = Newtonsoft.Json.JsonConvert.DeserializeObject<List<FabricVersion>>(json);
                Debug.WriteLine("fabric加载版本信息获取成功！");
             var fabricSubVersions = new ObservableCollection<FabricLoaderVersionViewModel>();//定义fabric版本集合
               // fabricSubVersions.Add(new FabricLoaderVersionViewModel() { version = "1.16.5" });
                //版本添加示例
                  foreach (var version in Fabricroot)
                   {
                   fabricSubVersions.Add(new FabricLoaderVersionViewModel() { version = version.version , minecraftVersion = MinecraftVersion });
                 }
               //获取版本
               modLoader.Add(new ModLoaderViewModel() { modLoaderType = "Forge" });
               modLoader.Add(new ModLoaderViewModel() { modLoaderType = "Fabric", subVersions = fabricSubVersions });
     
            }
            catch (Exception ex)
            {
                Debug.WriteLine("读取json失败：" + ex.Message);
            }
              }
    }
    //定义json结构
    public class FabricVersion
    {
        public string separator { get; set; }
        public int build { get; set; }
        public string maven { get; set; }
        public string version { get; set; }
        public bool stable { get; set; }
    }
}
```