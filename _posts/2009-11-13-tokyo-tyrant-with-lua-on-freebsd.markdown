---
title: Tokyo Tyrant + Lua + FreeBSD
layout: post
tags:
- freebsd
- tokyotyrant
- lua
- install
---
This post is first and foremost a reminder to me of how I got something working, so I'll be able to do it again in six months time. Hopefully it might help out somebody else if they're ever in the same position (unlikely, but possible).

The scenario: installing Tokyo Tyrant from source (not the ports tree) on FreeBSD 7.2 with the Lua extensions (the Lua installation IS from the ports tree, and is the lua port, which as of "now" is 5.1.something).

Tokyo Cabinet needs to be installed first, but we'll take that as read - it's not problematic.

The first thing to note is that if you've just installed Lua from the port, <code>lua</code> and <code>luac</code> will return command not found in whatever shell you're using. That's a bit annoying, but is easily fixed, as it's only because the port puts the executables in <code>/lua51</code> under <code>/usr/local/bin</code>, presumably to avoid version clashes. However, it would be lovely if it put in some default symlinks if none existed, so let's add some:

<code>
ln -s /usr/local/bin/lua51/lua /usr/local/bin/lua<br />
ln -s /usr/local/bin/lua51/luac /usr/local/bin/luac
</code>

You'll probably need to <code>sudo</code> to those if you're not running as root (and why would you be?).

Now, when you've got the latest Tokyo Tyrant source downloaded and extracted, the command line you'll want to run to configure is

<code>
./configure --enable-lua --with-lua=&lt;DIR&gt;
</code>

where <code>&lt;DIR&gt;</code> is a directory containing the <code>include</code> and <code>lib</code> directories for the lua headers, etc. Now, this is the slight issue, because the FreeBSD Lua port doesn't put these under the same root, and configure expects them to be. It's simple to fix though, with something like this...

<code>
mkdir ~/lua_temp<br />
ln -s /usr/local/include/lua51 ~/lua_temp/include<br />
ln -s /usr/local/lib/lua51 ~/lua_temp/lib<br />
cd &lt;tokyo tyrant source directory&gt;<br />
./configure --enable-lua --with-lua=~/lua_temp<br />
rm -rf ~/lua_temp<br />
</code>

Et voila, that should have picked up what it needed. As an aside, if you're on FreeBSD and you're compiling this stuff from source, you'll often find you want to use gmake not make, as the BSD make chokes on various bits of makefiles (the Tokyo * projects are good examples).
