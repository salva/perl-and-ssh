# Perl and SSH

Salvador Fandiño (sfandino@yahoo.com)

YAPC::Europe 2013

---

# How do you automate SSH tasks?

---

# If you use ...

- `system("ssh $host $cmd @args")`
- or backticks
- Net::SSH
- Net::SSH::Perl
- Expect
- Net::SSH::Expect

---

# ...you are living on the past!

---

# Using system or backticks

- ok for simple things
- but very inneficient, opens a new SSH connection every time
- no password authentication
- you have to properly quote the arguments
- sometimes twice
- or be insecure

---

# Net::SSH

- simple wrapper for the `ssh` command
- provides also `sshopen2` and `sshopen3`
- has same problems as using `system`

---

# Net::SSH::Perl

- *huge* module
- at its time it represented a great effort
- I guess most of it is a direct translation of OpenSSH C code
- but today, nobody maintains it
- lots of known [bugs](https://rt.cpan.org/Public/Dist/Display.html?Name=Net-SSH-Perl)
- very difficult to install (Math::Pari)
- API good for simple things, bad for advanced things
- supports SFTP via [Net::SFTP](https://metacpan.org/module/Net::SFTP), doesn't support SCP

---

# Expect

- Expect is a great module for automating interactive programs
- It's a good way to automate `ssh` password authentication
- After that, it is (usually) the wrong tool
- It talks to the remote shell, you don't want that!
- On some systems (AIX, HP-UX) ttys may drop data

---

# BTW...

    !perl
    my $pty = IO::Pty->new;
    my $expect = Expect->init($pty);
    my $pid = open2($in, $out, '-');
    unless ($pid) {
      defined $pid or die "unable to fork";
      $pty->make_slave_controlling_terminal;
      do { exec @ssh_cmd };
      exit -1;
    }

---

# Net::SSH::Expect

- Builds on top of Expect
- On the wrong way
- It talks to the shell
- It is not reliable

---

# So, what should I use?

---

# Modern SSH modules

- Net::SSH2
- Net::OpenSSH
- Net::SSH::Any

---

# Well, actually...

- Net::SSH2
- Net::OpenSSH
- Net::SSH::Any
- Net::SSH::Mechanize
- Net::OpenSSH::Parallel
- Net::OpenSSH::Compat
- Net::SFTP::Foreign
- ...

---

# Net::SSH2

---

# Net::SSH2

- wrapper for the libssh2 C library
- quite portable
- quite easy to install on Unix/Linux, almost easy to install on Windows, don't known about VMS
- it is a very thin wrapper
- efficient, can run several commands over one SSH connection
- C'ish low level API:
    - very simple things are easy to do
    - not so simple things become quite hard
- supports SCP
- very primitive and inneficient support for SFTP
- project started by David B. Robins, currently being actively maintained by Rafael Kitover

---

# Net::SSH2 - usage

    !perl
    use Net::SSH2;
 
    my $ssh2 = Net::SSH2->new();
 
    $ssh2->connect('example.com') or die $!;
 
    if ($ssh2->auth_keyboard('fizban')) {
        my $chan = $ssh2->channel();
        $chan->exec('program');
    }

---

# Net::SSH2 problems

- very low level C'ish API: not for lazy people
- libssh2 is not a mature project yet
- requires a C compiler

---

# Net::OpenSSH

---

# Net::OpenSSH

- wrapper around OpenSSH `ssh`
- uses its connection multiplexing feature
    - several commands can be run over the same connection
    - efficient
- Perlish API with lots of belts and whistles
- easy to use, complex things are almost easy
- can work asynchronously
- supports SFTP, SCP, rsync and sshfs
- automatic argument quoting

---

# Net::OpenSSH - usage

    !perl
    use Net::OpenSSH;
    my $ssh = Net::OpenSSH->new($host, user => $user, password => $password);
    $ssh->die_on_error("unable to connect");

    my @output = $ssh->capture("cat /etc/passwd");

    my ($out, $err) = $ssh->capture2("find /");

    $ssh->system({stdin_data => "hello\n"},
                 "cat >>log");

    my $pid = $ssh->spawn({stderr_to_stdout => 1,
                           stdout_file => "tar.log"},
                          'tar', 'cf', '/tmp/my backup.tar', '/home/me');
    waitpid($pid,0);

    $ssh->scp_get('/tmp/*.tar', '.');
    
    $ssh->rsync_put({verbose => 1, safe_links => 1},
                    "etc", "/etc");

    my $sftp = $ssh->sftp; # return a Net::SFTP::Foreign object
    my $ls = $sftp->ls;

---

# Net::OpenSSH argument quoting

    !perl

    my $out = $ssh->capture("ls /etc"); # no quoting
    my $out = $ssh->capture(ls => '~/My Documents'); # quoting
    my $out = $ssh->capture({quote_args => 0},
                            ls => '~/My Documents'); # no quoting

    # selectively quoting arguments:
    my $out = $ssh->capture(ls => \'~/My Documents/*.pdf');
            # lets file name wildcards be expanded by the remote shell

    my $out = $ssh->capture(ls => \\'~/My Documents/*.pdf'); # wrong!

    my $out = $ssh->capture(@cmd1, \\'&&', @cmd2, \\'2>/dev/null');

---

# Net::OpenSSH argument quoting

- On the stable release, argument quoting expects a POSIX compatible shell on the remote side

- The development release has support for different quoting backends
      - POSIX (i.e. ksh, bash) and csh already there
      - maybe Windows/DOS backend in the future

---

# Net::OpenSSH problems

- Net::OpenSSH does not work on Windows
    - OpenSSH multiplexing feature does not work on Windows, not even under cygwin
- Requires the OpenSSH `ssh` command

---

# Net::SSH::Any

---

# Net::SSH::Any

- API very similar to Net::OpenSSH
- works on top of
    - Net::SSH2
    - Net::OpenSSH
    - maybe Net::SSH::Perl in the future
    - maybe simple wrapper around native `ssh`

- supports SCP and SFTP for file transfers, efficiently
- a work in progress
- though, basic functionality is already stable

---

# Net::SSH::Mechanize

---

# Net::SSH::Mechanize

- It uses the AnyEvent framework
- It aims to support `sudo`ing smoothly
- Limited API
- It talks to the remote shell, unreliable!

---

# Net::OpenSSH::Parallel

---

# Net::OpenSSH::Parallel

- Run commands in parallel in remote hosts through SSH
- Build on top of Net::OpenSSH
- Declarative API:
    - tell the module all the actions you want to perform on the remote hosts
    - let the module take care of everything and do the tasks
    - handle possible errors

---

# Net::OpenSSH::Parallel - usage

    !perl
    my $pssh = Net::OpenSSH::Parallel->new;

    # tell the object what the remote hosts are:
    $pssh->add_host('host1', user => foo, password => $pwd);
    $pssh->add_host('host2', user => foo, password => $pwd);

    # declare the actions you want to run:
    $pssh->push('host1', command => "echo hello from host1");
    $pssh->push('host2', command => "echo hello from host2");

    # for several host in one call:
    $pssh->push('host*', command => "echo hello from some host");

    # with variable expansion:
    $pssh->push('host*', command => "echo hello from host %HOST%");

    # and run it:
    $pssh->run;

---

# Net::OpenSSH::Parallel - usage

    !perl
    # other actions:
    $pssh->push('*', scp_get => '/var/log/messages.0', 'logs/messages.0.%HOST%');

    $pssh->push('*', rsync_put => '/var/www', '/var/www');

    # can pass extra arguments to the underlaying Net::OpenSSH methods
    $pssh->push('*', rsync_put => { safe_links => 1,
                                    stdout_file => ['>>', 'mylog'],
                                    stderr_to_stdout => 1 },
				  '/var/www', '/var/www');

    # runs a custom sub on a locally forked process
    $pssh->push('*', parsub => \&my_sub);

---

# Net::OpenSSH::Parallel - Expect and sudo

    !perl
    sub sudo_install {
        my ($label, $ssh, @pkgs) = @_;
        my ($pty) = $ssh->open2pty('sudo', 'apt-get', 'install', @pkgs);
        my $expect = Expect->init($pty);
        $expect->raw_pty(1);
        $expect->expect($timeout, ":");
        $expect->send("$passwd\n");
        $expect->expect($timeout, "\n");
        $expect->raw_pty(0);
        while(<$expect>) { print };
        close $expect;
    }
 
    $pssh->push('*', parsub => \&sudo_install, 'scummvm');

---

# Net::OpenSSH::Parallel - adding dependencies

    !perl
    $pssh->push('dmz*', scp_get => '/etc/passwd', 'passwd.%HOST%');
    $pssh->push('safe', join => 'dmz*');
    $pssh->push('safe', command => mkdir, "/var/safe/$date");
    $pssh->push('safe', scp_put => 'passwd.*', "/var/safe/$date");
    $pssh->run;

---

# Net::OpenSSH::Parallel problems

- Unable to distribute tasks around a set of hosts
- Can't run more than one command per host simultaneously
- stdin/stdout/stderr has to go to the file system, no pipes between tasks

---

# Net::OpenSSH::Compat

---

# Net::OpenSSH::Compat

- Implements most of Net::SSH::Perl and Net::SSH2 APIs in top of Net::OpenSSH
- Because sometimes people has problems installing them

---

# Net::SFTP::Foreign

---

# Other interesting modules

- SSH::Batch
- GRID::Machine
- POE::Component::OpenSSH
- App::MrShell

---

# Questions

---

# Thank you!

---

# Links

- [The talk](http://github.com/salva/perl-and-ssh.git)
