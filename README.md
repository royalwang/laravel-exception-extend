laravel-exception-extend
==================

- 异常化编程拓展

---

- [使用前的准备（异常化表单验证）](#1)
- [使用方法](#2)
- [自定义验证消息](#3)
- [特殊用法（直接抛出异常）](#4)
- [实际项目中的应用节选](#5)
- [模型的异常化](#6)
- [额外的支持 - 模型观察者（模型事件监听）](#7)
- [模型事件的触发顺序](#8)

> 合理的利用异常化编程方法，可以使代码可维护性大幅提高。

<a name="1"></a>
## 使用前的准备

在 composer.json 文件中申明依赖：

```json
"five-say/laravel-exception-extend": "1.*"
```

在 `/app/config/app.php` 中设置“别名”

```php
'aliases' => array(
    ...
    'Validation' => 'FiveSay\ValidationFacade', // 异常化表单验证
),
```

<a name="2"></a>
## 使用方法

```php
try {
    Validation::make(array(
        'name' => 'required',
    ));
    
    // 验证通过后的其它操作 ...

} catch (FiveSay\ValidationException $e) {
    return Redirect::back()->withErrors($e->errors);
}
```

<a name="3"></a>
## 自定义验证消息

```php
Validation::make(
    array(
        'name' => 'required',
    ),
    array(
        'name.required' => '自定义验证消息',
    )
);
```

<a name="4"></a>
## 特殊用法（直接抛出异常）

```php
Validation::throwIt('name', 'test error message.');
```

```php
Validation::throwIt(array(
    'name'  => 'test error message.',
    'email' => 'test error message.',
));
```

<a name="5"></a>
## 实际项目中的应用节选

> 我们可以不受干扰的写完一整个业务逻辑，然后再针对捕获的异常做细化的错误处理，你会感觉到写代码从未有过的顺畅，整个代码的可读性也大幅度的提高。

```php
    /**
     * 创建
     * @return Response
     */
    public function store()
    {
        try {
            # 表单验证
            Validation::make(array(
                'account'          => 'required|between:3,50|unique:users',
                'password'         => 'required|between:5,32',
                'password_confirm' => 'required|same:password',
                'name'             => 'required|min:2',
                'mobiles'          => 'multi_mobile',
            ));

            # 创建使用者账号
            $user = User::create(
                array('activated' => true) // 强制激活
                + Input::only('account', 'password', 'name')
            )->setGroupTo('Reception');

            # 创建员工信息
            $staff = Staff::create(
                array(
                    'user_id' => $user->id,
                    'model'   => 'Reception',
                )
                + Input::only('name', 'mobiles')
            );

            # 创建前台
            $reception = Reception::create(
                array(
                    'user_id'  => $user->id,
                    'staff_id' => $staff->id,
                )
                + Input::only('name')
            );

            # 操作成功
            return Redirect::route('home')->withSuccess('操作成功');

        } catch (FiveSay\ValidationException $e) {
            return Redirect::back()->withErrors($e->errors);
        } catch (UserSaveFailException $e) {
            return Redirect::back()->withError('账号信息写入失败');
        } catch (StaffSaveFailException $e) {
            return Redirect::back()->withError('员工信息写入失败');
        } catch (ReceptionSaveFailException $e) {
            return Redirect::back()->withError('前台信息写入失败');
        }
    }
```

<a name="6"></a>
## 模型的异常化

> 上例中的 `UserSaveFailException` 即是模型异常化的结果。要达到这种效果仅仅需要一个步骤。让我们以 `User` 模型为例：

继承 `FiveSay\Model`（不必担心，我们的 `FiveSay\Model` 已经继承了 `Eloquent`）：

```php
class User extends FiveSay\Model
{ ...
```

<a name="7"></a>
## 额外的支持 - 模型观察者（模型事件监听）

> 继承了 `FiveSay\Model` 的模型将额外获得一个特性：当对应的“模型观察者类”（`XxxxObserver`）存在时，将自动载入。  
> 同样以 `User` 模型为例：当你在系统任何地方定义了 `UserObserver` 后，这个类就会被自动注册为“模型观察者”。

```php
class UserObserver {

    public function saving($model)
    {
        //
    }

    public function saved($model)
    {
        //
    }

}
```

<a name="8"></a>
## 模型事件的触发顺序

```php
<?php
/*
|--------------------------------------------------------------------------
| 模型观察者
|--------------------------------------------------------------------------
| 模型事件触发顺序
|--------------------------------------------------------------------------
|
| 创建 & 更新
|          |-- creating -- created --|
| saving --|                         |-- saved
|          |-- updating -- updated --|
| 
| 软删除 & 强制删除
|            |-- softing -- softed --|
| deleting --|                       |-- deleted
|            |-- forcing -- forced --|
| 
| 恢复
| restoring -- saving -- updating -- updated -- saved -- restored
| 
*/
class DemoObserver
{
    // ...
}
```

> **注意：** `softing` `softed` `forcing` `forced` 并非原生事件，是由 `FiveSay\Model` 拓展得到的。