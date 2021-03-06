README_FILE = 'README.md'
VENV_DIR = '.venv'
TENSORFLOW_URL = 'https://storage.googleapis.com/tensorflow/linux/cpu/tensorflow-0.12.0-cp35-cp35m-linux_x86_64.whl'
IN_VENV = ". #{VENV_DIR}/bin/activate &&"


def task_in_venv name, packages=[TENSORFLOW_URL], &block
  task name => %i(clean venv) do |t|
    def vsh *args, **kwargs
      sh([IN_VENV, *args.map{ |x| x.to_s }].join(' '),
         **kwargs)
    end

    vsh "pip3 install --upgrade #{packages.join ' '}"
    vsh 'python3 setup.py install'
    block.call t
  end
end


task :venv do
  sh "python3 -m venv #{VENV_DIR}" unless File.directory? VENV_DIR
end


task :clean do
  sh 'git clean -dfx'
end


task_in_venv :module_test do
  Dir.glob('qnd/**/*_test.py').each do |file|
    vsh 'pytest', file
  end
end


task_in_venv :script_test do
  vsh('python3 test/empty.py')

  Dir.glob('test/*.py').each do |file|
    vsh 'python3', file, '-h'
  end

  # Worker hosts should not include a master host.
  vsh(*%w(! python3 test/oracle.py
          --master_host localhost:4242
          --worker_host localhost:4242
          --ps_hosts localhost:5151
          --task_type job
          --train_file README.md
          --eval_file setup.py))
end


%i(mnist_simple mnist_multi_workers).each do |name|
  task_in_venv name do
    vsh "cd examples/#{name} && ./main.sh"
  end
end


task_in_venv :mnist_full do |t|
  [
    nil,
    %i(use_eval_input_fn),
    %i(use_dict_inputs),
    %i(use_model_fn_ops),
    %i(self_batch),
    %i(self_filename_queue use_eval_input_fn),
  ].each do |flags|
    [
      'clean',
      (flags and flags.map{ |flag| "#{flag}=yes"}.join(' ')),
    ].each do |args|
      vsh "make -C examples/#{t.name} #{args}"
    end
  end
end


task :test => %i(
  module_test
  script_test
  mnist_simple
  mnist_multi_workers
  mnist_full
)


task_in_venv :readme_examples do
  md = File.read(README_FILE)

  File.write(README_FILE, %(
#{md.match(/(\A.*## Examples)/m)[0]}

```python
#{File.read('examples/mnist_simple/mnist_simple.py').strip}
```

With the code above, you can create a command with the following interface.

```
#{`#{IN_VENV} python3 examples/mnist_simple/mnist_simple.py -h`.strip}
```

Explore [examples](examples) directory for more and see how to run them.


#{md.match(/## Caveats.*\Z/m)[0].strip}
).lstrip)
end


task_in_venv :html, ['pdoc', TENSORFLOW_URL] do
  vsh 'pdoc --html --html-dir docs --overwrite qnd'
end


task :doc => %i(html readme_examples)


task_in_venv :format, %w(autopep8) do
  sh "autopep8 -i #{Dir.glob('**/*.py').join ' '}"
end


task :upload => %i(test clean) do
  sh 'python3 setup.py sdist bdist_wheel'
  sh 'twine upload dist/*'
end
