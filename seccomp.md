# Benchmarking

Jump to [Results](#results):

Simple experiment showed seccomp-based syscall ~5 times slower than vanila one. 

Calling `write` syscall directly:
```C
const unsigned count = UINT_MAX / 10000;
	unsigned i = 0;
	for (i=0;i<count;++i) {
		syscall(__NR_write, STDERR_FILENO, buffer, sizeof(buffer)-1);
	}
```

Result:

    ➜  seccomp  time ./main 2> /dev/null
    ./main 2> /dev/null  0.05s user 0.05s system 96% cpu 0.107 total
    
Calling write trough trap mechamism:

```C
// trap handler
static void emulator(int nr, siginfo_t *info, void *void_context)
{
	ucontext_t *ctx = (ucontext_t *)(void_context);
	ctx->uc_mcontext.gregs[REG_RESULT] = syscall(__NR_write, STDERR_FILENO, buffer, sizeof(buffer)-1);
	return;
}

int main(int argc, char **argv)
{
	const unsigned count = UINT_MAX / 10000;
	unsigned i = 0;
	// calling write through trap mechamism
	for (i=0;i<count;++i) {
		syscall(__NR_read, STDERR_FILENO, buffer, sizeof(buffer)-1);
	}

	return 0;
}
```

Result is:

    ➜  seccomp  time ./main 2> /dev/null
    ./main 2> /dev/null  0.16s user 0.37s system 99% cpu 0.531 total

## Results

+ Having `BPF` filter set increase running time by ~2 (50ms vs 100ms)
+ Performing syscall via trap mechanism increase running time by ~5 (100ms vs 500ms)
    
# Using seccompsandbox

Playground utility from seccompsandbox (Chrome project) could be used to run some executable in a sandbox without source modification.
No recompilation, use custom LD_PRELOAD libs. Setup time takes quite long (1.7s on my machine).

Results:

    ➜  seccompsandbox git:(master) ✗ time sudo ./playground ./testing > /dev/null
    In secure mode, now!
    sudo ./playground ./testing > /dev/null  1.70s user 0.01s system 98% cpu 1.726 total
    
And results with numerous `write`'s:

    ➜  seccompsandbox git:(master) ✗ time sudo ./playground ./testing > /dev/null
    In secure mode, now!
    sudo ./playground ./testing > /dev/null  1.74s user 0.06s system 98% cpu 1.826 total
    
## What is it:

+ Uses `SECCOMP_MODE_FILTER` instead of `SECCOMP_MODE_STRICT` (only `read`, `write`, `exit`, `sigreturn` are prohibited)
+ Uses separate _trusted_ process to communicate with untrusted via sockets
+ Actual filtering occurs via custom handler for each syscall in trusted process
+ During setup looks for loaded libraries and patches all system call to point to their handlers

__It's broker + loader.__
    









