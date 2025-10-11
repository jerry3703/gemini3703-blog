---
title: lsktp_fabric通用写法（1.21.8）
published: 2025-09-29
description: "寻光开发组内部资料"
tags: ["dev","寻光开发组","JAVA","MC"]
category: 项目规划
draft: false
author: "黄子郬"
pinned: true
---
```java
package lsk.lsktp;

import com.mojang.brigadier.arguments.DoubleArgumentType;
import com.mojang.brigadier.arguments.StringArgumentType;
import com.mojang.brigadier.context.CommandContext;
import com.mojang.brigadier.exceptions.CommandSyntaxException;
import net.fabricmc.api.ModInitializer;
import net.fabricmc.fabric.api.command.v2.CommandRegistrationCallback;
import net.minecraft.command.argument.DimensionArgumentType;
import net.minecraft.entity.Entity;
import net.minecraft.registry.RegistryKey;
import net.minecraft.server.command.CommandManager;
import net.minecraft.server.command.ServerCommandSource;
import net.minecraft.server.world.ServerWorld;
import net.minecraft.text.Text;
import net.minecraft.world.World;

import java.util.HashSet;

public class Lsktp implements ModInitializer {


    @Override
    public void onInitialize() {

        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            dispatcher.register(CommandManager.literal("lsktp")
                    .then(CommandManager.argument("x", DoubleArgumentType.doubleArg())
                            .then(CommandManager.argument("y", DoubleArgumentType.doubleArg())
                                    .then(CommandManager.argument("z", DoubleArgumentType.doubleArg())
                                            //   .executes(context -> teleportPlayer(context))
                                            .executes(context ->execute_command(context.getSource(),context))
                                    )
                            )
                    )
                    .requires(source -> source.hasPermissionLevel(0)) // 设置权限等级
            );
        });
        CommandRegistrationCallback.EVENT.register((dispatcher, registryAccess, environment) -> {
            dispatcher.register(CommandManager.literal("lsktp")
                    .then(CommandManager.argument("d", DimensionArgumentType.dimension())
                            .then(CommandManager.argument("x", DoubleArgumentType.doubleArg())
                                    .then(CommandManager.argument("y", DoubleArgumentType.doubleArg())
                                            .then(CommandManager.argument("z", DoubleArgumentType.doubleArg())
                                                    //   .executes(context -> teleportPlayer(context))
                                                    .executes(context ->execute(context.getSource(),context))
                                            )
                                    )
                            )
                    )
                    .requires(source -> source.hasPermissionLevel(0)) // 设置权限等级
            );
        });

    }

    static int execute_command(ServerCommandSource Source, CommandContext<ServerCommandSource> context){
        Entity entity =  Source.getPlayer() ;

        double x = DoubleArgumentType.getDouble(context, "x");
        double y = DoubleArgumentType.getDouble(context, "y");
        double z = DoubleArgumentType.getDouble(context, "z");

        entity.teleport(Source.getWorld(),x , y, z, new HashSet<>(), entity.getYaw(), entity.getPitch(),false);//此处1.21.8 - 可能有更改，false可能没有
//      return execute_command(Source, Radius, Player, Origin.getPos());
        context.getSource().sendFeedback(() -> Text.literal("传送至 " + x + ", " + y + ", " + z), false);

        return 1;
    }
    private static int execute(ServerCommandSource Source, CommandContext<ServerCommandSource> context) throws CommandSyntaxException {
        ServerCommandSource source = context.getSource();
        Entity player = source.getPlayer();
        if (player == null) {
            source.sendError(Text.translatable("需玩家执行！"));
            return 0;
        }
        double x = DoubleArgumentType.getDouble(context, "x");
        double y = DoubleArgumentType.getDouble(context, "y");
        double z = DoubleArgumentType.getDouble(context, "z");
        ServerWorld d = DimensionArgumentType.getDimensionArgument(context, "d");

        RegistryKey<World> registryKey;

        registryKey = player.getWorld().getRegistryKey();

        ServerWorld targetWorld = player.getServer().getWorld(registryKey);
       // String ds = StringArgumentType.getString(context,"d");
        if (targetWorld != null) {
            player.teleport(d, x, y, z, new HashSet<>(), player.getYaw(), player.getPitch(), false);//此处1.21.8 - 可能有更改，false可能没有
            // source.sendFeedback("aaa", dimension), false);
            // context.setSuccessCount(1);
            context.getSource().sendFeedback(() -> Text.literal("传送至 "+ x + ", " + y + ", " + z), false);

            return 1;
        }


        return 1;
    }
}
```