# Shikashi - A flexible sandbox for ruby

Shikashi is an sandbox for ruby that handles all ruby method calls executed in the interpreter to allow or deny
these calls depending on the receiver object, the method name, the source file from where the call was originated
and the source file where the called method is implemented.

The permissions for each sandboxed run is fully configurable and the implementation of the methods called from within
the sandbox can be replaced transparently

The implementation of shikashi is written in pure ruby and now implemented based in evalhook, (see http://tario.github.com/evalhook)

## Installation

### Gem installation

Run in the terminal:

```
sudo gem install shikashi
```

OR

* Download the last version of the gem from http://github.com/tario/shikashi/downloads
* Install the gem with the following;

```
sudo gem install shikashi-X.X.X.gem.
```

### Troubleshooting

```
ERROR:  While executing gem ... (Gem::DependencyError)
    Unable to resolve dependencies: ruby2ruby requires sexp_processor (~> 3.0); ruby_parser requires sexp_processor (~> 3.0)
```

The version of ruby2ruby and ruby_parser required depends on sexp_processor 3.X but for some reason this version of the gem
is not automatically installed by gem, you can workaround this issue by installing it before using:

```
gem install sexp_processor --version '~> 3.2'
```

## Documentation

Full API documentation can be found on:
http://tario.github.com/shikashi/doc/

## Usage

This examples and more can be found in examples directory

### Basic Example

Hello world from a sandbox

```ruby
	require "rubygems"
	require "shikashi"

	include Shikashi

	s = Sandbox.new
	priv = Privileges.new
	priv.allow_method :print

	s.run(priv, 'print "hello world\n"')
```

### Basic Example 2

Call external method from inside the sandbox

```ruby
	require "rubygems"
	require "shikashi"

	include Shikashi

	def foo
		# privileged code, can do any operation
		print "foo\n"
	end

	s = Sandbox.new
	priv = Privileges.new

	# allow execution of foo in this object
	priv.object(self).allow :foo

	# allow execution of method :times on instances of Fixnum
	priv.instances_of(Fixnum).allow :times

	#inside the sandbox, only can use method foo on main and method times on instances of Fixnum
	s.run(priv, "2.times do foo end")
```

### Basic Example 3

Define a class outside the sandbox and use it in the sandbox

```ruby
	require "rubygems"
	require "shikashi"

	include Shikashi

	s = Sandbox.new
	priv = Privileges.new

	# allow execution of print
	priv.allow_method :print

	class X
		def foo
			print "X#foo\n"
		end

		def bar
			system("echo hello world") # accepted, called from privileged context
		end

		def privileged_operation( out )
			# write to file specified in out
			system("echo privileged operation > " + out)
		end
	end
	# allow method new of class X
	priv.object(X).allow :new

	# allow instance methods of X. Note that the method privileged_operations is not allowed
	priv.instances_of(X).allow :foo, :bar

	priv.allow_method :=== # for exception handling
	#inside the sandbox, only can use method foo on main and method times on instances of Fixnum
	s.run(priv, '
	x = X.new
	x.foo
	x.bar

	begin
	x.privileged_operation # FAIL
	rescue SecurityError
	print "privileged_operation failed due security error\n"
	end
	')
```

### Basic Example 4

define a class from inside the sandbox and use it from outside

```ruby
	require "rubygems"
	require "shikashi"

	include Shikashi

	s = Sandbox.new
	priv = Privileges.new

	# allow execution of print
	priv.allow_method :print

	#inside the sandbox, only can use method foo on main and method times on instances of Fixnum
	s.run(priv, '
	class X
		def foo
			print "X#foo\n"
		end

		def bar
			system("ls -l")
		end
	end
	')

	x = s.base_namespace::X.new
	x.foo
	begin
		x.bar
	rescue SecurityError => e
		print "x.bar failed due security errors: #{e}\n"
	end
```


### Base namespace

```ruby
	require "rubygems"
	require "shikashi"

	include Shikashi

	class X
		def foo
			print "X#foo\n"
		end
	end

	s = Sandbox.new

	s.run( "
	  class X
		def foo
			print \"foo defined inside the sandbox\\n\"
		end
	  end
	  ", Privileges.allow_method(:print))
	  

	x = X.new # X class is not affected by the sandbox (The X Class defined in the sandbox is SandboxModule::X)
	x.foo

	x = s.base_namespace::X.new
	x.foo
	
	s.run("X.new.foo", Privileges.allow_method(:new).allow_method(:foo))
```

### Timeout example

```ruby
	require "rubygems"
	require "shikashi"

	s = Shikashi::Sandbox.new
	perm = Shikashi::Privileges.new

	perm.allow_method :sleep

	s.run(perm,"sleep 3", :timeout => 2) # raise Shikashi::Timeout::Error after 2 seconds
```

## Copying

Copyright (c) 2010-2011 Dario Seminara, released under the GPL License (see LICENSE)
