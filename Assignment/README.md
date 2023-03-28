Assignment of OS
<code>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/shm.h>
#include <sys/wait.h>
  
#define BOARD_SIZE 100  
#define NUM_GENERATIONS 100  
#define NUM_PROCESSES 4  
  
// Function to initialize the game board with random values
void init_board(int *board) {
    for (int i = 0; i < BOARD_SIZE * BOARD_SIZE; i++) {
        board[i] = rand() % 2;
    }
}

// Function to print the game board
void print_board(int *board) {
    for (int i = 0; i < BOARD_SIZE; i++) {
        for (int j = 0; j < BOARD_SIZE; j++) {
            printf("%c ", board[i * BOARD_SIZE + j] ? '*' : '-');
        }
        printf("\n");
    }
}

// Function to apply the rules of the game to a single cell
int apply_rules(int *board, int x, int y) {
    int num_neighbors = 0;
    for (int i = x - 1; i <= x + 1; i++) {
        for (int j = y - 1; j <= y + 1; j++) {
            if (i == x && j == y) {
                continue;
            }
            if (i < 0 || i >= BOARD_SIZE || j < 0 || j >= BOARD_SIZE) {
                continue;
            }
            num_neighbors += board[i * BOARD_SIZE + j];
        }
    }
    if (board[x * BOARD_SIZE + y] == 1) {
        if (num_neighbors == 2 || num_neighbors == 3) {
            return 1;
        } else {
            return 0;
        }
    } else {
        if (num_neighbors == 3) {
            return 1;
        } else {
            return 0;
        }
    }
}

// Function to simulate a portion of the game board
void simulate_board(int *board, int start_row, int end_row, int *pipe_fd) {
    int msg = 0;
    for (int gen = 0; gen < NUM_GENERATIONS; gen++) {
        // Apply the rules of the game to each cell in the sub-grid
        for (int i = start_row; i < end_row; i++) {
            for (int j = 0; j < BOARD_SIZE; j++) {
                int new_val = apply_rules(board, i, j);
                board[i * BOARD_SIZE + j] = new_val;
            }
        }
        // Send a message to the parent process indicating that this process has completed one iteration
        write(pipe_fd[1], &msg, sizeof(msg));
        // Wait for a message from the parent process to proceed to the next iteration
        read(pipe_fd[0], &msg, sizeof(msg));
    }
    // Send a message to the parent process indicating that this process has completed all iterations
    write(pipe_fd[1], &msg, sizeof(msg));
}

int main() {
    int *board;
    int board_size = BOARD_SIZE * BOARD_SIZE;
    int shmid = shmget(IPC_PRIVATE, board_size * sizeof(int), IPC_CREAT | 0666);
    board = (int *) shmat(shmid, NULL, 0);
    init_board(board);
    int pipe_fd[NUM_PROCESSES][2];
    pid_t pid[NUM_PROCESSES];
    int rows_per_process = BOARD_SIZE / NUM_PROCESSES;
   

  // Create pipes for inter-process communication
for (int i = 0; i < NUM_PROCESSES; i++) {
    if (pipe(pipe_fd[i]) == -1) {
        fprintf(stderr, "Error creating pipe\n");
        return 1;
    }
}

// Fork child processes and execute Game of Life algorithm
for (int i = 0; i < NUM_PROCESSES; i++) {
    pid[i] = fork();
    if (pid[i] == 0) {
        // Child process
        int start_row = i * rows_per_process;
        int end_row = (i + 1) * rows_per_process - 1;
        if (i == NUM_PROCESSES - 1) {
            // Last process handles any remaining rows
            end_row = BOARD_SIZE - 1;
        }
        close(pipe_fd[i][0]); // Close read end of pipe
        game_of_life(board, start_row, end_row, pipe_fd[i][1]);
        close(pipe_fd[i][1]); // Close write end of pipe
        exit(0);
    } else if (pid[i] < 0) {
        fprintf(stderr, "Error forking child process\n");
        return 1;
    }
}

// Parent process waits for all child processes to finish
for (int i = 0; i < NUM_PROCESSES; i++) {
    waitpid(pid[i], NULL, 0);
}

// Detach and remove shared memory segment
shmdt(board);
shmctl(shmid, IPC_RMID, NULL);

return 0;
}
</code>
