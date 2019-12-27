NX是一个全栈工程集成框架，提供了一套完整的解决方案和工具来实现全栈项目开发。Nx使用了诸如Cypress,Jest,Prettier,TypeScript等现代化工具来实现包括Angular、React和Node、Nest、Express等各种框架的集成开发，并且提供了深度图、依赖检测等功能，使小型团队也能使用像Google,Facebook和微软一样的开发工具和流程。本文基于Nx官方入门案例，以实现对Nx的使用流程和特性的熟悉和入门。关于NX生成项目文档见[README_NX](./README_NX.md).

### 1. 创建一个新的nx工程
使用下面任意一种即可。注意国内安装cypress会很慢，最好在.npmrc文件中修改路径。
预设模板选择empty，选择Angular CLI作为命令行工具。这样就创建了一个空的工程。
```shell
npx create-nx-workspace@latest myworkspace
npm init nx-workspace myworkspace
yarn create nx-workspace myworkspace
cd myworkspace
```

### 2. 创建一个Angular应用

如果已经安装有Angular CLI（推荐），可以用ng命令快速开始
先用默认参数添加angular，然后创建一个名为todos的应用
```shell
ng add @nrwl/angular --defaults
ng g @nrwl/angular:application todos
```
命令会生成一个空的Angular应用和对应的E2E测试程序。
输入`ng serve todos`运行程序。

### 3. 添加测试

Nx默认使用Cypress运行E2E测试。
修改apps/todos-e2e/src/support/app.po.ts文件.
```TypeScript
export const getTodos = () => cy.get('li.todo');
export const getAddTodoButton = () => cy.get('button#add-todo');
```
然后修改apps/todos-e2e/src/integration/app.spec.ts

```TypeScript
import { getAddTodoButton, getTodos } from '../support/app.po';

describe('TodoApps', () => {
  beforeEach(() => cy.visit('/'));

  it('should display todos', () => {
    getTodos().should(t => expect(t.length).equal(2));
    getAddTodoButton().click();
    getTodos().should(t => expect(t.length).equal(3));
  });
});
```
停止`ng serve`，运行`ng e2e todos-e2e`或者`ng e2e todos-e2e --watch`启动测试程序，会打开一个UI界面。如果加上--headless后缀，则打开无头浏览器（不显示前端界面）。

### 4. 修改Angular程序以通过测试

打开apps/todos/src/app/app.component.ts文件。并修改如下：
```TypeScript
import { Component } from '@angular/core';

interface Todo {
  title: string;
}

@Component({
  selector: 'myorg-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  todos: Todo[] = [{ title: 'Todo 1' }, { title: 'Todo 2' }];

  addTodo() {
    this.todos.push({
      title: `New todo ${Math.floor(Math.random() * 1000)}`
    });
  }
}
```
打开apps/todos/src/app/app.component.html文件并修改如下：
```html
<h1>Todos</h1>

<ul>
  <li *ngFor="let t of todos" class="todo">{{ t.title }}</li>
</ul>

<button id="add-todo" (click)="addTodo()">Add Todo</button>
```
重复运行前一步的测试，可以看到测试已经通过。(如果前一步打开watch选项，则测试会自动重新运行)。

### 5. 连接API服务

导入HttpClientModule来使用Http服务。在apps/todos/src/app/app.module.ts中添加Import语句，并且在@NgModule声明的import部分加以声明。
```TypeScript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';
import { HttpClientModule } from '@angular/common/http';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HttpClientModule],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule {}
```
修改apps/todos/src/app/app.component.ts文件，使用http服务：
```TypeScript
import { Component } from '@angular/core';
import { HttpClient } from '@angular/common/http';

interface Todo {
  title: string;
}

@Component({
  selector: 'myorg-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  todos: Todo[] = [];

  constructor(private http: HttpClient) {
    this.fetch();
  }

  fetch() {
    this.http.get<Todo[]>('/api/todos').subscribe(t => (this.todos = t));
  }

  addTodo() {
    this.http.post('/api/addTodo', {}).subscribe(() => {
      this.fetch();
    });
  }
}
```
运行ng serve，因为此时服务端还不能提供api接口服务，因此页面看不到任何数据。

### 6. 添加后端API服务接口

本例使用Nest框架提供后端服务接口，这是一个非常优秀的后端框架。在各种后端项目中得到广泛应用。
使用下面的命令生成一个api服务，在待输入的目录选项中直接回车即可（不需要分api目录级别）
```shell
ng g @nrwl/nest:app api --frontendProject=todos
```
在工作空间的apps目录下面可以看到前端工程文件夹todos，后端工程文件夹api，以及前端测试工程todos-e2e，等等。在nx前后端一体的工程中，通过这样的文件架构，nx可以在同一个工作空间管理和执行前端、后端和测试工程的各个环节。
可以巡幸下列命令来执行后端项目。
- `ng serve api`运行后端应用
- `ng build api`编译后端应用
- `ng test api`运行后端测试
打开apps/api/src/app/app.service.ts文件并添加服务。
```TypeScript
import { Module } from '@nestjs/common';

import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService]
})
export class AppModule {}
```
打开apps/api/src/app/app.controller.ts文件并添加服务。
```TypeScript
import { Controller, Get, Post } from '@nestjs/common';

import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get('todos')
  getData() {
    return this.appService.getData();
  }

  @Post('addTodo')
  addTodo() {
    return this.appService.addTodo();
  }
}
```
再打开命令行输入http://localhost:3333/api/todos就可以看到默认的todos内容。如果使用VSCode可以用Rest Client插件测试POST http://localhost:3333/api/addtodo功能。或者使用curl或者Postman等其他测试工具均可。POST之后再Get，会发现新的Todo条目已经添加进去了。


### 7. 设置代理

在前一步添加api服务的时候，我们使用了--frontendProject=todos的参数。这个参数实际上添加了供Angular在开发时访问服务端的能力。其中的魔法在根目录下的angular.json文件中。
```json
{
  "serve": {
    "builder": "@angular-devkit/build-angular:dev-server",
    "options": {
      "browserTarget": "todos:build",
      "proxyConfig": "apps/todos/proxy.conf.json"
    },
    "configurations": {
      "production": {
        "browserTarget": "todos:build:production"
      }
    }
  }
}
```
其中`proxyConfig`条目声明了代理的位置。打开该文件：
```json
{
  "/api": {
    "target": "http://localhost:3333",
    "secure": false
  }
}
```
这里把所有/api的请求自动转向至api后端。这时候如果分别在两个终端里输入`ng serve api`和`ng serve todos`，再打开网址`http://127.0.0.1:4200`，会发现todos应用已经可以正常工作了。

### 8. 代码共享

目前，程序前后端工作良好，但Nx的框架特点并没有体现出来，Todo接口在前后端同时声明，随着项目扩展，前后端的衔接会逐渐出现问题，因此需要Nx框架来创建数据library来集成数据。
在根目录下运行`ng g @nrwl/workspace:lib data`生成data目录。

打开/libs/data/src/lib/data.ts，定义Todo接口。
```TypeScript
export interface Todo {
  title: string;
}
```
分别在后端文件apps/api/src/app/app.service.ts和前端文件apps/todos/src/app/app.component.ts中分别引入lib中的Todo接口来替换之前的声明。
```TypeScript
import { Todo } from '@myorg/data' ;
```

### 9.创建共享库(lib)
Nxde的共享库不仅仅可以用来共享代码，更可以通过工厂模式通过公开API接口将代码组织为更好的小模块。
#### UI界面库
此处通过创建一个Angular组件来说明。运行`ng g @nrwl/angular:lib ui`来创建一个anular ui共享库。
指令会创建libs/ui/库，在libs/ui/src/lib/ui.module.ts文件中会初始化创建如下代码:
```shell
ng g @nrwl/angular:lib ui
```

```TypeScript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';

@NgModule({
  imports: [CommonModule]
})
export class UiModule {}
```
#### 创建一个组件

接下来通过如下命令在ui库中创建一个angular 组件todos。
```shell
ng g component todos --project=ui --export
```
像在angular中一样修改生成的libs/ui/src/lib/todos/todos.component.ts文件和todos.component.html文件。注意这里也用到了上一节创建的Todo接口。
```TypeScript
//todos.component.ts
import { Component, OnInit, Input } from '@angular/core';
import { Todo } from '@myorg/data';

@Component({
  selector: 'myorg-todos',
  templateUrl: './todos.component.html',
  styleUrls: ['./todos.component.css']
})
export class TodosComponent implements OnInit {
  @Input() todos: Todo[];

  constructor() {}

  ngOnInit() {}
}
```

```html
//todos.component.html
<ul>
 <li *ngFor="let t of todos">{{t.title}}</li>
 </ui>
```

#### 使用创建的 UI库

用生成的ui库替换之前创建的组件。
更新apps/todos/src/app/app.module.ts来引入组件。
```TypeScript
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';
import { HttpClientModule } from '@angular/common/http';
import { UiModule } from '@myorg/ui';

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule, HttpClientModule, UiModule],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule {}
```
更新app.component.html来使用组件。
```html
<h1>Todos</h1>

<myorg-todos [todos]="todos"></myorg-todos>

<button (click)="addTodo()">Add Todo</button>
```

### 生成系统架构图Dep Graph

一个大型工程可能包含成百上千个组件和应用，很快这些组件之间的依赖就会变得错综复杂难以理解。Nx为此提供了一个专门用于生成组件关系图的命令行工具Dep Graph。可以让开发人员对组件关系一目了然。运行下列命令。Nx会打开浏览器http://127.0.0.1:4211生成系统架构图。
```shell
npm run dep-graph
```
本项目生成的架构图如下：


![DepGraph.png](:/ad733cfc0cbc4d66af53d9f3e3a175ae)

### 检查依赖影响

除了生成依赖图表，Nx还可以检查项目中相互影响关系的变动。在尝试该步骤前需要与git配合，如果系统中没有安装git需要先安装。此外，由于本项目是从git仓库中下载来的，因此如果要推送到远程仓库需要先重置远程仓库目标来退送到自己的仓库中。
执行下列命令提交到目前为止的变更。

```shell
git add .
git commit -am 'init'
```

接下来修改程序源代码，比如在libs/ui/src/lib/todos/todos.component.html文件的</li>符号前加一个错误改为 !</li>。
然后运行下列命令，系统会列出affected：apps todos.表示该修改影响了依赖组件todos。

```shell
npm run affected:apps
```
如果输入下面命令，会显示受到影响的共享库（这里是ui）。
```shell
npm run affected:libs
```
#### 影响测试

除了手动运行上述命令检查变动牵连到了哪些组件和共享库之外，也可以运行影响测试，来检查改动是否对测试产生影响（还能不能通过测试）。因为我们修改了程序之后一直没有修改对应的测试文件，所以这里运行是通不过测试的。

```shell
npm run affected:test
```

运行影响测试命令会重新开始运行测试程序，并列举出测试问题，如果只想看到没有通过测试的内容，可以传入`--only-failed`参数。

#### 同步测试

为了加快测试速度，可以传入`--parallel`参数来使用同步技术加快测试速度。

#### 变动构建(build)

代码变动后可进行变动构建以快速构建变动部分。

```shell
npm run affected:build #变动构建
npm run affected -- -- target=build #和上一命令等价
```

### 汇总

通过本项目实现了以下工作。
- 使用前端框架Angular和后端框架Nest实现全栈工程开发
- 在前端和后端工程之间实现了代码共享
- 创建了一个UI库
- 使用Nx的深度图工具来生成项目依赖关系图
- 通过Nx的依赖检测工具实现了依赖变动的检测
