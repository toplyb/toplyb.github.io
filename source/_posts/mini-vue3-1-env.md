---
title: mini Vue3（1） - 环境搭建
date: 2025-08-08 19:10:56
tags:
   - 前端
   - Vue
categories:
   - 技术
---

# 环境搭建

## 搭建 monorepo 环境

1. 初始化项目

   `pnpm init`

2. 新增 pnpm 配置文件

   `.npmrc`

   Pnpm 安装扩展后，会将该扩展下所有依赖的包都放在自己的目录下面，而不是 `node_modules` 根目录下，增加下面的配置，可将依赖的包都提升到 `node_modules` 根目录下，这样可以解决一些依赖查找的问题

   如果有多个扩展依赖同一个包，但是各自依赖的包的版本不一致，`pnpm` 这时不会将该扩展提升到根目录下，仍然是放在各自扩展的目录中

   ```shell
   shamefully-hoist=true
   ```

   `pnpm-workspace.yaml`

   指明 monorepo 环境中项目的目录

   ```shell
   packages:
     - 'packages/**'
   ```

注意事项：使用上面步骤即可搭建成 `monorepo` 环境，此时，如果想要安装扩展的话，如 `pnpm install vue` 的话，则会报错
此时需要指明是将扩展安装在哪个位置，是在最外层公共的还是说在 `packages` 中的某一个包下面，如果安装在公共中的话，则可以使用 `pnpm install vue -w` 进行安装
如果是想要放在 `packages` 下的某一个包中的话，则使用该命令：`pnpm install @vue/shared --workspace --filter @vue/reactivity`，该命令的意思是将当前 `packages` 中的 `@vue/shared` 的包安装到当前 `packages` 下的 `@vue/reactivity` 中

## 搭建开发环境

1. 安装依赖

   `Vue3` 使用 `TypeScript` 编写，所以需要安装 `typescript`，`esbuild` 用来打包项目，`minimist` 用来解析命令行

   `pnpm install typescript esbuild minimist -d -w`

2. 配置 tsconfig

   `tsconfig.json`

   ```json
   {
     "compilerOptions": {
       "outDir": "dist",
       "sourceMap": true,
       "target": "ES2016",
       "module": "esnext",
       "moduleResolution": "node",
       "strict": false,
       "resolveJsonModule": true,
       "esModuleInterop": true,
       "jsx": "preserve",
       "lib": ["esnext", "dom"],
       "baseUrl": ".",
       "paths": {
         "@vue/*": ["packages/*/src"]
       }
     },
     "include": ["packages/**/*.ts", "packages/**/*.d.ts", "packages/**/*.tsx", "packages/**/*.vue"],
     "exclude": ["node_modules", "packages/**/node_modules"]
   }
   ```

3. 项目打包

   `package.json`

   在 `scripts` 对象中新增打包命令，其中 `reactivity` 表示打包的模块，`-f esm` 表示要打包的格式，`-f` 后面可以选择 `esm`、`commonjs` 等

   ```json
   {
     "name": "vue3-resource",
     "private": true,
     "version": "1.0.0",
     "description": "",
     "main": "index.js",
     "type": "module",
     "scripts": {
       "dev": "node scripts/dev.js reactivity -f esm"
     },
     ...
   }
   ```

   `/script/dev.js`

   ```js
   import minimist from 'minimist';
   import { resolve, dirname } from 'path';
   import { fileURLToPath } from 'url';
   import { createRequire } from 'module';
   import esbuild from 'esbuild';
   
   // 获取 package.json 中配置的打包的命令中的参数
   const args = minimist(process.argv.slice(2));
   const __filename = fileURLToPath(import.meta.url);
   const __dirname = dirname(__filename);
   const require = createRequire(import.meta.url);
   
   // 打包的模块的名称
   const target = args._[0] || 'reactivity';
   // 打包的格式
   const format = args.f || 'iife';
   // 打包的入口
   const entry = resolve(__dirname, `../packages/${target}/src/index.ts`);
   // 各个模块的依赖及各自信息
   const pkg = require(`../packages/${target}/package.json`);
   
   esbuild.context({
     // 入口
     entryPoints: [entry],
     // 打包输出的路径
     outfile: resolve(__dirname, `../packages/${target}/dist/${target}.js`),
     // 将各自模块的依赖和模块打包到一起
     bundle: true,
     // 打包后提供给浏览器使用
     platform: 'browser',
     sourcemap: true,
     format,
     globalName: pkg.buildOptions?.name
   }).then((ctx) => {
     return ctx.watch();
   });
   ```
