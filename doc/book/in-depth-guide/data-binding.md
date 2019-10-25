# 编辑以及删除数据

在之前的文档中我们学习了如何使用 zend-form 和 zend-db 组件来 *创建* 一个新的数据集。这篇文档将通过介绍 *编辑* 和 *删除* 数据来完善CURD功能。

## 将对象绑定到表单上

“新增内容”和“编辑内容”唯一的区别就在于表单中存在的数据不一样。这将意味着我们可以将数据库中获取到的数据直接填充到表单上。幸运的是，zend-form 提供了一个 **data-binding** 的功能。

为了使用这个功能，你需要将一个`Post`实例并将其绑定到表单上。实现方式如下：
- 在`WriteController`中新增依赖`PostRepositoryInterface`以检索我们的`Post`。
- 在`WriteController`中添加一个`editAction`方法，用来将检索到的`Post`绑定到表单，并且展示或进行处理。
- 在`WriteControllerFactory`中注入`PostRepositoryInterface`。

开始更新`WriteController`:

- 引入`PostRepositoryInterface`。
- 添加一个变量来保存`PostRepositoryInterface`。
- 在构造函数中添加 `PostRepositoryInterface`。
- 添加 `editAction`函数实例。

最终效果将如下所示：

```php
<?php
// In module/Blog/src/Controller/WriteController.php:

namespace Blog\Controller;

use Blog\Form\PostForm;
use Blog\Model\Post;
use Blog\Model\PostCommandInterface;
use Blog\Model\PostRepositoryInterface;
use InvalidArgumentException;
use Zend\Mvc\Controller\AbstractActionController;
use Zend\View\Model\ViewModel;

class WriteController extends AbstractActionController
{
    /**
     * @var PostCommandInterface
     */
    private $command;

    /**
     * @var PostForm
     */
    private $form;

    /**
     * @var PostRepositoryInterface
     */
    private $repository;

    /**
     * @param PostCommandInterface $command
     * @param PostForm $form
     * @param PostRepositoryInterface $repository
     */
    public function __construct(
        PostCommandInterface $command,
        PostForm $form,
        PostRepositoryInterface $repository
    ) {
        $this->command = $command;
        $this->form = $form;
        $this->repository = $repository;
    }

    public function addAction()
    {
        $request   = $this->getRequest();
        $viewModel = new ViewModel(['form' => $this->form]);

        if (! $request->isPost()) {
            return $viewModel;
        }

        $this->form->setData($request->getPost());

        if (! $this->form->isValid()) {
            return $viewModel;
        }

        $post = $this->form->getData();

        try {
            $post = $this->command->insertPost($post);
        } catch (\Exception $ex) {
            // An exception occurred; we may want to log this later and/or
            // report it to the user. For now, we'll just re-throw.
            throw $ex;
        }

        return $this->redirect()->toRoute(
            'blog/detail',
            ['id' => $post->getId()]
        );
    }

    public function editAction()
    {
        $id = $this->params()->fromRoute('id');
        if (! $id) {
            return $this->redirect()->toRoute('blog');
        }

        try {
            $post = $this->repository->findPost($id);
        } catch (InvalidArgumentException $ex) {
            return $this->redirect()->toRoute('blog');
        }

        $this->form->bind($post);
        $viewModel = new ViewModel(['form' => $this->form]);

        $request = $this->getRequest();
        if (! $request->isPost()) {
            return $viewModel;
        }

        $this->form->setData($request->getPost());

        if (! $this->form->isValid()) {
            return $viewModel;
        }

        $post = $this->command->updatePost($post);
        return $this->redirect()->toRoute(
            'blog/detail',
            ['id' => $post->getId()]
        );
    }
}
```

`addAction` 和 `editAction` 最主要的不同点在于后者需要先获取一个`Post`，并将其*绑定*到表单上。
通过这种绑定的方式，确保数据填充并初始化表单以用来展示数据，一单数据发生变更，实例也讲同步更新。
这将意味着我们在对表单进行验证后不必再去调用`getData`方法。

现在需要更新`WriteControllerFactory`。添加一个如下的引用：

```php
// In module/Blog/src/Factory/WriteControllerFactory.php:
use Blog\Model\PostRepositoryInterface;
```

接下来更新工厂类的内容如下：

```php
// In module/Blog/src/Factory/WriteControllerFactory.php:
public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
{
    $formManager = $container->get('FormElementManager');

    return new WriteController(
        $container->get(PostCommandInterface::class),
        $formManager->get(PostForm::class),
        $container->get(PostRepositoryInterface::class)
    );
}
```

上面我们同时更新了控制器和模型，接下来我们需要修改路由。

## 添加用于编辑的路由

用于编辑的路由和之前定义的 `blog` 路由类似，但是有一下两个不同的地方：

- 需要拥有一个 `edit` 前缀
- 并路由到 `WriteController`

在'blog' 的 `child_routes` 中添加如下所示的新路由：

```php
// In module/Blog/config/module.config.php:

use Zend\Router\Http\Segment;

return [
    'service_manager' => [ /* ... */ ],
    'controllers'     => [ /* ... */ ],
    'router'          => [
        'routes' => [
            'blog' => [
                /* ... */

                'child_routes' => [
                    /* ... */

                    'edit' => [
                        'type' => Segment::class,
                        'options' => [
                            'route'    => '/edit/:id',
                            'defaults' => [
                                'controller' => Controller\WriteController::class,
                                'action'     => 'edit',
                            ],
                            'constraints' => [
                                'id' => '[1-9]\d*',
                            ],
                        ],
                    ],
                ],
            ],
        ],
    ],
    'view_manager'    => [ /* ... */ ],
];
```

## 创建编辑模板
## Creating the edit template

`add` 和 `edit` 模板实际的渲染结果本质上是一样的；二者唯一不同就在于表单的action属性不一样。
因此，我们将为表单创建一个新的*partial*脚本，更新`add`视图并加入当前脚本，然后我们再创建一个`edit`视图模板。
创建一个内容如下所示的新文件`module/Blog/view/blog/write/form.phtml`:

```php
<?php
$form = $this->form;
$fieldset = $form->get('post');

$title = $fieldset->get('title');
$title->setAttribute('class', 'form-control');
$title->setAttribute('placeholder', 'Post title');

$text = $fieldset->get('text');
$text->setAttribute('class', 'form-control');
$text->setAttribute('placeholder', 'Post content');

$submit = $form->get('submit');
$submit->setValue($this->submitLabel);
$submit->setAttribute('class', 'btn btn-primary');

$form->prepare();

echo $this->form()->openTag($form);
?>

<fieldset>
<div class="form-group">
    <?= $this->formLabel($title) ?>
    <?= $this->formElement($title) ?>
    <?= $this->formElementErrors()->render($title, ['class' => 'help-block']) ?>
</div>

<div class="form-group">
    <?= $this->formLabel($text) ?>
    <?= $this->formElement($text) ?>
    <?= $this->formElementErrors()->render($text, ['class' => 'help-block']) ?>
</div>
</fieldset>

<?php
echo $this->formSubmit($submit);
echo $this->formHidden($fieldset->get('id'));
echo $this->form()->closeTag();
```

现在我们参照如下的内容更新`add`视图模板，`module/Bolg/view/write/add.phtml`:

```php
<h1>Add a blog post</h1>

<?php
$form = $this->form;
$form->setAttribute('action', $this->url());
echo $this->partial('blog/write/form', [
    'form' => $form,
    'submitLabel' => 'Insert new post',
]);
```

我们对以上获取到的表单设置了action属性值，且为提交按钮填充了一个满足上下文条件的文本，并使用我们上面编写的partial视图脚本进行了渲染。

接下来我们来创建一个新的视图脚本，`blog/write/edit`：

```php
<h1>Edit blog post</h1>

<?php
$form = $this->form;
$form->setAttribute('action', $this->url('blog/edit', [], true));
echo $this->partial('blog/write/form', [
    'form' => $form,
    'submitLabel' => 'Update post',
]);
```

`add` 和 `edit` 模板有如下三点不同：


- 页面顶部的标题不一致。
- action属性传入的链接不一致。
- 提交按钮的文本不一致。

因为URI需要传入一个ID, 为了确保这个id的合法性，我们在控制器中将id作为一个参数传入：`$this->url('blog/edit/', ['id' => $id])`。
然而zend-router同时也允许'增加一些别的参数，但是我们可以通过将view-helper的最后一个参数设置为true的方式来告知其为最终的结果。

如果你此时刷新页面，你将得到如下的错误信息：
```
Call to member function getId() on null
```
那是因为我们还没有在类中实现更新函数来返回一个Post对象。接下来我们开始操作。

编辑文件`module/Blog/src/Model/ZendDbSqlCommand.php`，按照如下方式更新`updatePost`方法：

```php
public function updatePost(Post $post)
{
    if (! $post->getId()) {
        throw new RuntimeException('Cannot update post; missing identifier');
    }

    $update = new Update('posts');
    $update->set([
            'title' => $post->getTitle(),
            'text' => $post->getText(),
    ]);
    $update->where(['id = ?' => $post->getId()]);

    $sql = new Sql($this->db);
    $statement = $sql->prepareStatementForSqlObject($update);
    $result = $statement->execute();

    if (! $result instanceof ResultInterface) {
        throw new RuntimeException(
            'Database error occurred during blog post update operation'
        );
    }

    return $post;
}
```

他看起来和我们之前实现的`insertPost`类似。主要的不同是我们使用了`Update`类；不再使用 `values()` 方法，转而使用：

- `set()`，传入需要更新的值。
- `where()`, 确定需要更新那条记录（当前需要获取单条记录）.

此外我们还会在执行操作之前测试id是否存在，因为实际已经存在了，所以当我们将`Post` 的修改提交到数据库后其也讲如实的返回所有的数据。

## 实现删除功能

最后但并非不重要的，是需要删除一些数据。首先我们在`ZendDbSqlCommand` 类中实现一个 `deletePost` 方法：

```php
// In module/Blog/src/Model/ZendDbSqlCommand.php:

public function deletePost(Post $post)
{
    if (! $post->getId()) {
        throw new RuntimeException('Cannot update post; missing identifier');
    }

    $delete = new Delete('posts');
    $delete->where(['id = ?' => $post->getId()]);

    $sql = new Sql($this->db);
    $statement = $sql->prepareStatementForSqlObject($delete);
    $result = $statement->execute();

    if (! $result instanceof ResultInterface) {
        return false;
    }

    return true;
}
```

在上面的方法中，我们使用`Zend\Db\Sql\Delete` 来创建用于删除指定id的SQL，然后执行。

接下来，我们参照如下内容在新文件`module/Blog/src/Controller/DeleteController.php`
中创建一个新的控制器 `Blog\Controller\DeleteController`:

```php
<?php
namespace Blog\Controller;

use Blog\Model\Post;
use Blog\Model\PostCommandInterface;
use Blog\Model\PostRepositoryInterface;
use InvalidArgumentException;
use Zend\Mvc\Controller\AbstractActionController;
use Zend\View\Model\ViewModel;

class DeleteController extends AbstractActionController
{
    /**
     * @var PostCommandInterface
     */
    private $command;

    /**
     * @var PostRepositoryInterface
     */
    private $repository;

    /**
     * @param PostCommandInterface $command
     * @param PostRepositoryInterface $repository
     */
    public function __construct(
        PostCommandInterface $command,
        PostRepositoryInterface $repository
    ) {
        $this->command = $command;
        $this->repository = $repository;
    }

    public function deleteAction()
    {
        $id = $this->params()->fromRoute('id');
        if (! $id) {
            return $this->redirect()->toRoute('blog');
        }

        try {
            $post = $this->repository->findPost($id);
        } catch (InvalidArgumentException $ex) {
            return $this->redirect()->toRoute('blog');
        }

        $request = $this->getRequest();
        if (! $request->isPost()) {
            return new ViewModel(['post' => $post]);
        }

        if ($id != $request->getPost('id')
            || 'Delete' !== $request->getPost('confirm', 'no')
        ) {
            return $this->redirect()->toRoute('blog');
        }

        $post = $this->command->deletePost($post);
        return $this->redirect()->toRoute('blog');
    }
}
```

同  `WriterController`,，其同样需要引入 `PostRepositoryInterface` 和 `PostCommandInterface`。
前者用于确保我们能引用有效的post实例，后者确保我们能执行删除操作。


当用户使用 `GET` 方法请求页面时，我们将展示一个包含当前post内容的页面，并提供一个确认表单。
提交后，我们将会检查他们已经确认并允许了当前删除操作。如果执行失败或者执行成功，我们都将跳转到文章列表页。

和别的控制器一样，我们也需要实现一个工厂方法。
创建文件 `module/Blog/src/Factory/DeleteControllerFactory` 并填充上如下内容：

```php
<?php
namespace Blog\Factory;

use Blog\Controller\DeleteController;
use Blog\Model\PostCommandInterface;
use Blog\Model\PostRepositoryInterface;
use Interop\Container\ContainerInterface;
use Zend\ServiceManager\Factory\FactoryInterface;

class DeleteControllerFactory implements FactoryInterface
{
    /**
     * @param ContainerInterface $container
     * @param string $requestedName
     * @param null|array $options
     * @return DeleteController
     */
    public function __invoke(ContainerInterface $container, $requestedName, array $options = null)
    {
        return new DeleteController(
            $container->get(PostCommandInterface::class),
            $container->get(PostRepositoryInterface::class)
        );
    }
}
```

我们需要确保告知应用工厂和控制器的映射关系，并且提供一个新的路由。打开文件 `module/Blog/config/module.config.php` 按照如下步奏编辑。

首先，建立控制器和工厂的映射：

```php
'controllers' => [
    'factories' => [
        Controller\ListController::class => Factory\ListControllerFactory::class,
        Controller\WriteController::class => Factory\WriteControllerFactory::class,
        // Add the following line:
        Controller\DeleteController::class => Factory\DeleteControllerFactory::class,
    ],
],
```

然后在"blog"路由中添加子路由：

```php
'router' => [
    'routes' => [
        'blog' => [
            /* ... */

            'child_routes' => [
                /* ... */

                'delete' => [
                    'type' => Segment::class,
                    'options' => [
                        'route' => '/delete/:id',
                        'defaults' => [
                            'controller' => Controller\DeleteController::class,
                            'action'     => 'delete',
                        ],
                        'constraints' => [
                            'id' => '[1-9]\d*',
                        ],
                    ],
                ],
            ],
        ],
    ],
],
```
 最后，创建一个新的视图模板`module/Blog/view/blog/delete/delete.phtml`，内容如下：

```php
<h1>Delete post</h1>

<p>Are you sure you want to delete the following post?</p>

<ul class="list-group">
    <li class="list-group-item"><?= $this->escapeHtml($this->post->getTitle()) ?></li>
</ul>

<form action="<?php $this->url('blog/delete', [], true) ?>" method="post">
    <input type="hidden" name="id" value="<?= $this->escapeHtmlAttr($this->post->getId()) ?>" />
    <input class="btn btn-default" type="submit" name="confirm" value="Cancel" />
    <input class="btn btn-danger" type="submit" name="confirm" value="Delete" />
</form>
```

这次，我们并没有使用zend-form; 因为他只需要提供一个隐藏的元素和取消/确认按钮，并不需要为其提供面向对象的模型。

现在，我们可以访问已存在的博客内容，例如`http://localhost:8080/blog/delete/1` 来查看删除表单。如果你点击`Cancel`将会返回列表页；如果你选择 `Delete` 按钮，当前文章将会被删除页面也将返回到列表页，同时你将会看见之前的文章也将不存在了。

## 让列表页变得更加实用

当前我们的文章列表页展示了包括文章内容在内的所有记录；同时也没有导航链接，
这就意味着我们每次测试方法的时候都要手动去更新浏览器地址栏的内容。接下来我们将其更新到更实用的列表：

- 仅仅展示文章的标题；
- 在文章标题上增加展示文章内容的链接；
- 提供编辑和删除文章的链接；
- 添加新增文章的按钮。


在实际应用中，我们可能需要通过某种权限控制方式来确定是否展示编辑以及删除链接；我们将会有别的教程来专门探讨这个问题。

打开文件 `module/Blog/view/blog/list/index.phtml`，并按照如下内容更新:

```php
<h1>Blog Posts</h1>

<div class="list-group">
<?php foreach ($this->posts as $post): ?>
  <div class="list-group-item">
    <h4 class="list-group-item-heading">
      <a href="<?= $this->url('blog/detail', ['id' => $post->getId()]) ?>">
        <?= $post->getTitle() ?>
      </a>
    </h4>

    <div class="btn-group" role="group" aria-label="Post actions">
      <a class="btn btn-xs btn-default" href="<?= $this->url('blog/edit', ['id' => $post->getId()]) ?>">Edit</a>
      <a class="btn btn-xs btn-danger" href="<?= $this->url('blog/delete', ['id' => $post->getId()]) ?>">Delete</a>
    </div>
  </div>    
<?php endforeach ?>
</div>

<div class="btn-group" role="group" aria-label="Post actions">
  <a class="btn btn-primary" href="<?= $this->url('blog/add') ?>">Write new post</a>
</div>
```

至此，我们拥有了一个可以使用链接和按钮在不同页面之间转换的功能强大的博客系统了。

## 总结
## Summary

在当前章节中，我们学习了如何使用 zend-form 组件的数据绑定功能，并使用其实现了更新功能。
同时也学习了如何将控制器和表单合理的解耦，帮助我们将控制器细枝末节的操作中脱离出来。

我们也尝试了去使用了视图结构化分离，分离出重复的视图并将其重用。尤其是防止出现一些重复的标识。

最后，我们使用了 `Zend\Db\Sql` 的两个子组件，学习了如何使用 `Update` 和 `Delete` 的功能。

在接下来的章节中，我们将对之前的所有操作进行总结。我们将讨论之前使用过的设计模式，并对在本章节中可能出现的几个问题进行讨论。
