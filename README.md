#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

int main() {
    pid_t p = fork();

    if (p < 0) {
        printf("Child process failed to create\n");
    } else if (p == 0) {
        printf("Child process has been created\n");
        printf("Parent process id: %d\n", getppid());
        printf("Child process id: %d\n", getpid());
    } else {
        printf("Parent process created\n");
        printf("Parent process id: %d\n", getpid());
        printf("Child process id: %d\n", p);
    }

    return 0;
}
