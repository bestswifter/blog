---


---

<p>假设我们有两个分支，a 和 b，它们的提交都有一个相同的父提交（master 指向的那次提交）。如图所示：</p>
<p><img src="https://diycode.b0.upaiyun.com/photo/2018/ee182be2c2bb3990544575a3a6e7dcfb.png" alt=""></p>
<p>现在我们在分支 a 上，然后 rabase 到分支 b 上。如图所示：</p>
<p><img src="https://diycode.b0.upaiyun.com/photo/2018/47e1df3203b276353d17e5102ed3d640.png" alt=""></p>
<p>平时开发中经常遇到这种情况，假设分支 a 和 b 是两个独立的 feature 分支，但是不小心被我们错误的 rebase 了。现在相当于两个 feature 分支中原本独立的业务被揉起来了，当然是我们不想看到的结果，那么如何撤销呢？</p>
<hr>
<p>一种方案是利用 reflog 命令。</p>
<h2 id="利用-reflog-撤销变基">利用 reflog 撤销变基</h2>
<p>我们先不考虑原理，直接上解决方案，首先输入 <code>git reflog</code>，你会看到如下图所示的日志：</p>
<p><img src="https://diycode.b0.upaiyun.com/photo/2018/657ea37b9ffa116f5d1615b021cbf2e8.png" alt=""></p>
<p>最后的输出其实是最早的操作，我们逐条分析下：</p>
<ol>
<li>HEAD@{8}: 这里我们创建了初始的提交</li>
<li>HEAD@{7}：检出了分支 a</li>
<li>HEAD@{6}：在分支 a 上做了一次提交，注意 master 分支没有变动</li>
<li>HEAD@{5}：从分支 a 回到分支 master，相当于向后退了一次</li>
<li>HEAD@{4}：检出了分支 b</li>
<li>HEAD@{3}：在分支 b 上做了一次提交，注意 master 分支没有变动</li>
<li>HEAD@{2}：这一步开始变基到分支 a，首先切换到分支 a 上</li>
<li>HEAD@{1}：把分支 b 对应的那次提交变基到分支 a 上</li>
<li>HEAD@{0}：变基结束，因为是在 b 上发起的变基，所以最后还切回分支 b</li>
</ol>
<p>如果我们想撤销此次 rebase，只要输入以下命令就可以了：</p>
<pre class=" language-shell"><code class="prism  language-shell">git reset --hard HEAD@{3}
</code></pre>
<p>此时再看，已经“恢复”到 rebase 前的状态了。的是不是感觉很神奇呢，先别着急，后面会介绍这么做的原理。</p>
<h2 id="git-工作原理简介">git 工作原理简介</h2>
<p>为了搞懂 git 是如何工作的，以及这些命令背后的原理，我想有必要对 git 的模型有基础的了解。</p>
<p>首先，每一个 git 目录都有一个名为 <code>.git</code> 的隐藏目录，关于 git 的一切都存储于这个目录里面（全局配置除外）。这个目录里面有一些子目录和文件，文件其实不重要，都是一些配置信息，后面会介绍其中的 HEAD 文件。子目录有以下几个：</p>
<ol>
<li>info：这个目录不重要，里面有一个 exclude 文件和 <code>.gitignore</code> 文件的作用相似，区别是这个文件不会被纳入版本控制，所以可以做一些个人配置。</li>
<li>hooks：这个目录很容易理解， 主要用来放一些 git 钩子，在指定任务触发前后做一些自定义的配置，这是另外一个单独的话题，本文不会具体介绍。</li>
<li>objects：用于存放所有 git 中的对象，下面单独介绍。</li>
<li>logs：用于记录各个分支的移动情况，下面单独介绍。</li>
<li>refs：用于记录所有的引用，下面单独介绍。</li>
</ol>
<p>本文主要会介绍后面三个文件夹的作用。</p>
<h2 id="git-对象">git 对象</h2>
<p>git 是面向对象的！
git 是面向对象的！
git 是面向对象的！</p>
<p>没错，git 是面向对象的，而且很多东西都是对象。我举个简单的例子，来帮助大家理解这个概念。假设我们在一个空仓库里，编辑了 2 个文件，然后提交。此时都会有那些对象呢？</p>
<p>首先会有两个数据对象，每个文件都对应一个<strong>数据对象</strong>。当文件被修改时，即使是新增了一个字母，也会生成一个新的数据对象。</p>
<p>其次，会有一个<strong>树对象</strong>用来维护一系列的数据对象，叫树对象的原因是它持有的不仅可以是数据对象，还可以是另一个树对象。比如某次提交了两个文件和一个文件夹，那么树对象里面就有三个对象，两个是数据对象，文件夹则用另一个树对象表示。这样递归下去就可以表示任意层次的文件了。</p>
<p>最后则是提交对象，每个提交对象都有一个树对象，用来表示某一次提交所涉及的文件。除此以外，每一个提交还有自己的父提交，指向上一次提交的对象。当然，提交对象还会包含提交时间、提交者姓名、邮箱等辅助信息，就不多说了。</p>
<p>假设我们只有一个分支，以上知识点就足够解释 git 的提交历史是如何计算的了。它并不存储完整的提交历史，而是通过父提交的对象不断向前查找，得出完整的历史。</p>
<p>注意开头那张图片，分支 b 指向的提交是 <code>9cbb015</code>，不妨来看下它是何方神圣：</p>
<pre class=" language-shell"><code class="prism  language-shell">git cat-file -t 9cbb015
git cat-file -p 9cbb015
</code></pre>
<p>这里我们使用 <code>cat-file</code> 命令，其中 <code>-t</code> 参数打印对象的类型，<code>-p</code> 参数会智能识别类型，并打印其中的内容。输出结果如图所示：</p>
<p><img src="https://diycode.b0.upaiyun.com/photo/2018/d6d696a50b7858b7155e6d61a8121bb6.png" alt=""></p>
<p>可见 <code>9cbb015</code> 是一个<strong>提交对象</strong>，里面包含了树对象、父提交对象和各种配置信息。我们可以再打印树对象看看：</p>
<p><img src="https://diycode.b0.upaiyun.com/photo/2018/d416d3f25ebe98b1fd2ffa91e9c204ce.png" alt=""></p>
<p>这表示本次提交只修改了 begin 这个文件，并且输出了 begin 这个文件对于的数据对象。</p>
<h2 id="git-引用">git 引用</h2>
<p>既然 git 是面向对象的，那么有没有指正呢？还真是有的，分支和标签都是指向提交对象的指针。这一点可以验证：</p>
<pre class=" language-shell"><code class="prism  language-shell">cat .git/refs/heads/a
</code></pre>
<p>所有的本地分支都存储在 <code>git/refs/heads</code> 目录下，每一个分支对应一个文件，文件的内容如图所示：</p>
<p><img src="https://diycode.b0.upaiyun.com/photo/2018/9f92541a984b319e5b7a7c3cc3d3452c.png" alt=""></p>
<p>可见，<code>4a3a88d</code> 刚好是本文第一张图中分支 a 所指向的提交。</p>
<p>我们已经搞明白了 git 分支的秘密，现在有了所有分支的记录，又有了每次提交的父提交对象，就能够得出像 SourceTree 或者文章开头第一张图那样的提交状态了。</p>
<p>至于标签，它其实也是一种引用，可以理解为不能移动的分支。只能永远指向某个固定的提交。</p>
<p>最后一个比较特殊的引用是 HEAD，它可以理解为指针的指针，为了证明这一点，我们看看 <code>.git/HEAD</code> 文件：</p>
<p><img src="https://diycode.b0.upaiyun.com/photo/2018/3bbb796b6688e9a044c00873e0a5ed57.png" alt=""></p>
<p>它的内容记录了当前指向哪个分支，<code>refs/heads/b</code> 其实是一个文件，这个文件的内容是分支 b 指向的那个提交对象。理解这一点非常重要，否则你会无法理解 <code>checkout</code> 和 <code>reset</code> 的区别。</p>
<p>这两个命令都会改变 HEAD 的指向，区别是 <code>checkout</code> 不改变 HEAD 指向的分支的指向，而 <code>reset</code> 会。举个例子， 在分支 b 上执行以下两个命令都会让 HEAD 指向 <code>4a3a88d</code> 这次提交（分支 a 指向的提交）：</p>
<pre class=" language-shell"><code class="prism  language-shell">git checkout a
git reset --hard a
</code></pre>
<p>但 <code>checkout</code> 仅改变 HEAD 的指向，不会改变分支 b 的指向。而 <code>reset</code> 不仅会改变 HEAD 的指向，还因为 HEAD 指向分支 <code>b</code>，就把 b 也指向 <code>4a3a88d</code> 这次提交。</p>
<h2 id="git-日志">git 日志</h2>
<p>在 <code>.git/logs</code> 目录中，有一个文件夹和一个 HEAD 文件，每当 HEAD 引用改变了指向的位置，就会在 <code>.git/logs/HEAD</code> 中添加了一个记录。而 <code>.git/logs/refs/heads</code> 这个目录中则有多个文件，每个文件对应一个分支，记录了这个分支 的指向位置发生改变的情况。</p>
<p>当我们执行 <code>git reflog</code> 的时候，其实就是读取了 <code>.git/logs/HEAD</code> 这个文件。</p>
<p><img src="https://diycode.b0.upaiyun.com/photo/2018/ecde8541da6dbf0e35b472eddc15ec92.png" alt=""></p>
<h2 id="撤销-rebase-的原理">撤销 rebase 的原理</h2>
<p>首先我们要排除一个误区，那就是 git 会维护每次提交的提交对象、树对象和数据对象，但并不会维护每次提交时，各个分支的指向。在介绍分支的那一节中我们已经看到，分支仅仅是一个保留了提交对象的文件而已，并不记录历史信息。即使在上一节中，我们知道分支的变化信息会被记录下来，但也不会和某个提交对象绑定。</p>
<p>也就是说，git 中<strong>并不存在某次提交时的分支快照</strong></p>
<p>那么我们是如何通过 reset 来撤销 rebase 的呢，这里还要澄清另一个事实。前文曾经说过，某个时刻下你通过 SourceTree 或者 <code>git log</code> 看到的分支状态，其实是由所有分支的列表、每个分支所指向的提交，和每个提交的父提交共同绘制出来的。</p>
<p>首先 <code>git/refs/heads</code> 下的文件告诉我们有多少分支，每个文件的内容告诉我们这个分支指向那个提交，有了这个提交不断向前追溯就绘制出了这个分支的提交历史。所有分子的提交历史也就组成了我们看到的状态。</p>
<p>但我们要明确：<strong>不是所有提交对象都能看到的</strong>，举个例子如果我们把某个分支向前移一次提交，那个分支的提交线就会少一个节点，如果没有别的提交线包含这个节点，这个节点就看不到了。</p>
<p>所以在 rebase 完成后，我们以为看到了下面这样的提交线：</p>
<pre><code>df0f2c5(master) --- 4a3a88d(a) --- 9cbb015(b)
</code></pre>
<p>实际上是这样的：</p>
<pre><code>df0f2c5(master) --- 4a3a88d(a) --- 9d0618e(b)
   |
9cbb015
</code></pre>
<p>master 分支上依然有分叉，原来 <code>9cbb015</code> 这次提交依然存在，只不过没有分支的提交线包含它，所以无法看到而已。但是通过 <code>reflog</code>，我们可以找回 HEAD 头的每一次移动，所以能看到这次提交。</p>
<p>当我们执行这个命令时：</p>
<pre class=" language-shell"><code class="prism  language-shell">git reset --hard HEAD@{3}
</code></pre>
<p>再看一次 <code>reflog</code> 的输出：</p>
<p><img src="https://diycode.b0.upaiyun.com/photo/2018/657ea37b9ffa116f5d1615b021cbf2e8.png" alt=""></p>
<p><code>HEAD@{3}</code> 其实是它左侧 <code>9cbb015</code> 这次提交的缩写，所以上述命令等价于：</p>
<pre class=" language-shell"><code class="prism  language-shell">git reset --hard 9cbb015
</code></pre>
<p>前文说过，<code>reset</code> 不仅会移动 HEAD，还会移动 HEAD 所指向的分支，所以这个命令的执行结果就是让 HEAD 和分支 b 同时指向 <code>9cbb015</code> 这个提交，看起来像是撤销了 rebase。</p>
<p>但别忘了，分支 a 的上面还是有一次提交的，9d0618e 这次提交仅仅是没有分支指向它，所以不显示而已。但它真实的存在着，<strong>严格意义上来说，我们并没有真正的撤销此次 rebase</strong>。</p>

