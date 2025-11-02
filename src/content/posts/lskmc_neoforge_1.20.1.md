---
title: lsktp_neoforge通用写法（1.20.1）
published: 2025-10-24
description: "寻光开发组内部资料"
tags: ["dev","寻光开发组","JAVA","MC"]
category: 项目规划
draft: false
author: "黄子郬"
pinned: true
---
```java
//CommandEventHandler.java
//抄前先把forge自带的示例全删了
//modid为lsktp
package com.lsktp.lsktp;

import com.mojang.brigadier.arguments.DoubleArgumentType;
import com.mojang.brigadier.builder.LiteralArgumentBuilder;
import com.mojang.brigadier.builder.RequiredArgumentBuilder;
import com.mojang.brigadier.context.CommandContext;
import com.mojang.brigadier.exceptions.CommandSyntaxException;
import net.minecraft.commands.CommandSourceStack;
import net.minecraft.commands.Commands;
import net.minecraft.commands.arguments.DimensionArgument;
import net.minecraft.network.chat.Component;
import net.minecraft.resources.ResourceKey;
import net.minecraft.server.level.ServerLevel;
import net.minecraft.server.level.ServerPlayer;
import net.minecraftforge.event.RegisterCommandsEvent;
import net.minecraftforge.eventbus.api.SubscribeEvent;
import net.minecraftforge.fml.common.Mod;

import java.util.function.Supplier;
import java.util.logging.Level;

@Mod.EventBusSubscriber(modid = "lsktp")
public class CommandEventHandler {
    @SubscribeEvent
    public static void onRegisterCommands(RegisterCommandsEvent event) {
        event.getDispatcher().register(
                LiteralArgumentBuilder.<CommandSourceStack>literal("lsktp")
                        .requires(source -> source.hasPermission(0))
                        .then(RequiredArgumentBuilder.<CommandSourceStack, Double>argument("x", DoubleArgumentType.doubleArg())
                                .then(RequiredArgumentBuilder.<CommandSourceStack, Double>argument("y", DoubleArgumentType.doubleArg())
                                        .then(RequiredArgumentBuilder.<CommandSourceStack, Double>argument("z", DoubleArgumentType.doubleArg())
                                                .executes(CommandEventHandler::teleportPlayer)
                                        )
                                )
                        )
        );

        event.getDispatcher().register(
                LiteralArgumentBuilder.<CommandSourceStack>literal("lsktp")
                        .requires(source -> source.hasPermission(0))
                        .then(Commands.argument("d", DimensionArgument.dimension())
                                .then(RequiredArgumentBuilder.<CommandSourceStack, Double>argument("x", DoubleArgumentType.doubleArg())
                                        .then(RequiredArgumentBuilder.<CommandSourceStack, Double>argument("y", DoubleArgumentType.doubleArg())
                                                .then(RequiredArgumentBuilder.<CommandSourceStack, Double>argument("z", DoubleArgumentType.doubleArg())
                                                        .executes(CommandEventHandler::teleportPlayer_d)
                                                )
                                        )
                                )
                        )
        );
}
    private static int teleportPlayer(CommandContext<CommandSourceStack> context) {
        ServerPlayer player;
        try {
            player = context.getSource().getPlayerOrException();
        } catch (CommandSyntaxException e) {
            context.getSource().sendFailure(Component.literal("只能由玩家执行此命令！"));
            return 0;
        }
        double x = DoubleArgumentType.getDouble(context, "x");
        double y = DoubleArgumentType.getDouble(context, "y");
        double z = DoubleArgumentType.getDouble(context, "z");

        // 在 1.20.x 中，使用 teleportTo(ServerLevel, x, y, z, yaw, pitch)
        ServerLevel currentLevel = (ServerLevel) player.level();
        player.teleportTo(currentLevel, x, y, z, player.getYRot(), player.getXRot());


        context.getSource().sendSuccess(() -> Component.literal("传送至 " + x + ", " + y + ", " + z), false);
       // player.sendSystemMessage(Component.literal("传送至 " + x + ", " + y + ", " + z));
        return 1;
    }

    private static int teleportPlayer_d(CommandContext<CommandSourceStack> context) throws CommandSyntaxException {
        ServerPlayer player;
        try {
            player = context.getSource().getPlayerOrException();
        } catch (CommandSyntaxException e) {
            context.getSource().sendFailure(Component.literal("只能由玩家执行此命令！"));
            return 0;
        }
        double x = DoubleArgumentType.getDouble(context, "x");
        double y = DoubleArgumentType.getDouble(context, "y");
        double z = DoubleArgumentType.getDouble(context, "z");

        // DimensionArgument 在较新版本中返回的是 ResourceKey<Level>
       // ResourceKey<Level> dimKey = DimensionArgument.getDimension(context, "d");
       // ServerLevel targetWorld = context.getSource().getServer().getLevel(dimKey);
        ServerLevel targetWorld = DimensionArgument.getDimension(context, "d");
        if (targetWorld == null) {
            context.getSource().sendFailure(Component.literal("目标维度不存在！"));
            return 0;
        }

        player.teleportTo(targetWorld, x, y, z, player.getYRot(), player.getXRot());

        context.getSource().sendSuccess(() -> Component.literal("传送至 " + x + ", " + y + ", " + z), false);
        return 1;
    }
}

···