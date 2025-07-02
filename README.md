#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <ctype.h>
#include <locale.h>
#include <stdbool.h>

#define MAX_WORD_LEN     100
#define MIN_WORD_LEN     3
#define MAX_PLAYER_NAME  50
#define MAX_HIGH_SCORES  10
#define COLOR_GREEN      "\033[1;32m"
#define COLOR_YELLOW     "\033[1;33m"
#define COLOR_BLUE       "\033[1;34m"
#define COLOR_RESET      "\033[0m"

typedef struct {
    char text[MAX_WORD_LEN+1];
    int length;
    int difficulty;
    bool found;
    int start_row;
    int start_col;
} Word;

typedef struct {
    char player_name[MAX_PLAYER_NAME];
    int score;
    time_t date;
} HighScore;

typedef struct {
    char **matrix;
    int size;
    Word *words;
    int word_count;
    int found_words;
    int attempts;
    int score;
    int difficulty;
} GameState;

// Protótipos de função
void init_game(GameState *game);
void cleanup_game(GameState *game);
int load_words(const char *filename, Word **words);
int save_words(const char *filename, Word *words, int count);
void start_new_game(GameState *game);
void show_main_menu();
void show_word_manager();
void show_high_scores();
void show_settings();
void setup_game_difficulty(GameState *game);
void generate_matrix(GameState *game);
int place_word(GameState *game, const char *word);
void show_color_matrix(const GameState *game);
void play_game(GameState *game);
int check_word(GameState *game, int start_row, int start_col, int end_row, int end_col);
void calculate_score(GameState *game);
void save_high_score(const GameState *game);
void load_high_scores(HighScore **scores, int *count);
void add_word_to_dict();
void edit_word_in_dict();
void remove_word_from_dict();
void list_words_in_dict();
int get_valid_int(const char *prompt, int min, int max);
void clear_input_buffer();
void print_centered(const char *text);
void print_header(const char *title);

// Variáveis globais
const char *WORD_FILE = "palavras.dat";
const char *SCORE_FILE = "scores.dat";

int main() {
    setlocale(LC_ALL, "pt_BR.UTF-8");
    srand((unsigned)time(NULL));
    
    GameState game;
    init_game(&game);
    
    show_main_menu();
    
    cleanup_game(&game);
    return 0;
}

void init_game(GameState *game) {
    memset(game, 0, sizeof(GameState));
    game->size = 0;
    game->matrix = NULL;
    game->words = NULL;
    game->word_count = 0;
}

void cleanup_game(GameState *game) {
    if(game->matrix) {
        for(int i = 0; i < game->size; i++) {
            free(game->matrix[i]);
        }
        free(game->matrix);
    }
    free(game->words);
}

int load_words(const char *filename, Word **words) {
    FILE *file = fopen(filename, "rb");
    if(!file) return 0;
    
    fseek(file, 0, SEEK_END);
    long file_size = ftell(file);
    rewind(file);
    
    int count = file_size / sizeof(Word);
    if(count == 0) {
        fclose(file);
        return 0;
    }
    
    *words = malloc(count * sizeof(Word));
    if(!*words) {
        fclose(file);
        return 0;
    }
    
    if(fread(*words, sizeof(Word), count, file) != count) {
        free(*words);
        fclose(file);
        return 0;
    }
    
    fclose(file);
    return count;
}

int save_words(const char *filename, Word *words, int count) {
    FILE *file = fopen(filename, "wb");
    if(!file) return 0;
    
    int result = fwrite(words, sizeof(Word), count, file) == count;
    fclose(file);
    return result;
}

void show_main_menu() {
    int choice;
    GameState game;
    init_game(&game);
    
    do {
        print_header("CAÇA-PALAVRAS");
        printf("1. Novo Jogo\n");
        printf("2. Gerenciar Palavras\n");
        printf("3. Recordes\n");
        printf("4. Configurações\n");
        printf("0. Sair\n");
        
        choice = get_valid_int("Escolha uma opção", 0, 4);
        
        switch(choice) {
            case 1: 
                start_new_game(&game); 
                cleanup_game(&game);
                init_game(&game);
                break;
            case 2: show_word_manager(); break;
            case 3: show_high_scores(); break;
            case 4: show_settings(); break;
            case 0: printf("Até logo!\n"); break;
            default: printf("Opção inválida!\n");
        }
    } while(choice != 0);
    
    cleanup_game(&game);
}

void start_new_game(GameState *game) {
    cleanup_game(game);
    init_game(game);
    
    setup_game_difficulty(game);
    
    game->word_count = load_words(WORD_FILE, &game->words);
    if(game->word_count < 5) {
        printf("Não há palavras suficientes no dicionário (mínimo 5).\n");
        return;
    }
    
    generate_matrix(game);
    play_game(game);
}

void setup_game_difficulty(GameState *game) {
    print_header("DIFICULDADE");
    printf("1. Fácil (7x7, 5 palavras)\n");
    printf("2. Médio (8x8, 7 palavras)\n");
    printf("3. Difícil (9x9, 10 palavras)\n");
    
    game->difficulty = get_valid_int("Escolha a dificuldade", 1, 3);
    
    switch(game->difficulty) {
        case 1:
            game->size = 7;
            game->word_count = 5;
            break;
        case 2:
            game->size = 8;
            game->word_count = 7;
            break;
        case 3:
            game->size = 9;
            game->word_count = 10;
            break;
    }
}

void generate_matrix(GameState *game) {
    game->matrix = malloc(game->size * sizeof(char*));
    for(int i = 0; i < game->size; i++) {
        game->matrix[i] = malloc(game->size * sizeof(char));
        for(int j = 0; j < game->size; j++) {
            game->matrix[i][j] = 'A' + rand() % 26;
        }
    }
    
    for(int i = 0; i < game->word_count; i++) {
        int word_index;
        do {
            word_index = rand() % game->word_count;
        } while(strlen(game->words[word_index].text) > game->size);
        
        if(!place_word(game, game->words[word_index].text)) {
            printf("Não foi possível posicionar a palavra: %s\n", game->words[word_index].text);
        }
    }
}

int place_word(GameState *game, const char *word) {
    int directions[8][2] = {{0,1}, {1,0}, {0,-1}, {-1,0}, {1,1}, {1,-1}, {-1,1}, {-1,-1}};
    int word_len = strlen(word);
    int attempts = 0;
    int max_attempts = game->size * game->size * 8;
    
    while(attempts < max_attempts) {
        int dir_index = rand() % 8;
        int row = rand() % game->size;
        int col = rand() % game->size;
        
        int dr = directions[dir_index][0];
        int dc = directions[dir_index][1];
        
        int end_row = row + dr * (word_len - 1);
        int end_col = col + dc * (word_len - 1);
        
        if(end_row < 0 || end_row >= game->size || end_col < 0 || end_col >= game->size) {
            attempts++;
            continue;
        }
        
        bool valid = true;
        for(int i = 0; i < word_len; i++) {
            int r = row + dr * i;
            int c = col + dc * i;
            char current = game->matrix[r][c];
            
            if(current != toupper(word[i]) && current != ' ') {
                valid = false;
                break;
            }
        }
        
        if(!valid) {
            attempts++;
            continue;
        }
        
        for(int i = 0; i < word_len; i++) {
            int r = row + dr * i;
            int c = col + dc * i;
            game->matrix[r][c] = toupper(word[i]);
        }
        
        for(int i = 0; i < game->word_count; i++) {
            if(strcmp(game->words[i].text, word) == 0) {
                game->words[i].start_row = row;
                game->words[i].start_col = col;
                game->words[i].found = false;
                break;
            }
        }
        
        return 1;
    }
    
    return 0;
}

void play_game(GameState *game) {
    game->found_words = 0;
    game->attempts = 0;
    
    while(game->found_words < game->word_count) {
        show_color_matrix(game);
        
        printf("\nPalavras restantes (%d/%d):\n", game->found_words, game->word_count);
        for(int i = 0; i < game->word_count; i++) {
            if(!game->words[i].found) {
                printf("- %s\n", game->words[i].text);
            }
        }
        
        printf("\n1. Tentar palavra\n");
        printf("2. Pedir dica\n");
        printf("3. Desistir\n");
        
        int choice = get_valid_int("Escolha", 1, 3);
        
        if(choice == 1) {
            printf("Digite as coordenadas (linha coluna linha coluna): ");
            int start_row, start_col, end_row, end_col;
            if(scanf("%d %d %d %d", &start_row, &start_col, &end_row, &end_col) != 4) {
                printf("Entrada inválida!\n");
                clear_input_buffer();
                continue;
            }
            
            if(start_row < 0 || start_row >= game->size || start_col < 0 || start_col >= game->size ||
               end_row < 0 || end_row >= game->size || end_col < 0 || end_col >= game->size) {
                printf("Coordenadas fora da matriz!\n");
                continue;
            }
            
            game->attempts++;
            if(check_word(game, start_row, start_col, end_row, end_col)) {
                printf(COLOR_GREEN "Parabéns! Você encontrou uma palavra!\n" COLOR_RESET);
            } else {
                printf("Nenhuma palavra encontrada nessas coordenadas.\n");
            }
        } else if(choice == 2) {
            printf("Dica: Primeira letra de uma palavra não encontrada está em (%d,%d)\n",
                   game->words[game->found_words].start_row, 
                   game->words[game->found_words].start_col);
        } else {
            printf("Jogo encerrado.\n");
            break;
        }
    }
    
    calculate_score(game);
}

void show_color_matrix(const GameState *game) {
    printf("\n   ");
    for(int j = 0; j < game->size; j++) printf("%2d ", j);
    putchar('\n');
    
    for(int i = 0; i < game->size; i++) {
        printf("%2d ", i);
        for(int j = 0; j < game->size; j++) {
            if(game->matrix[i][j] >= '0' && game->matrix[i][j] <= '9') {
                printf(COLOR_GREEN " %c " COLOR_RESET, game->matrix[i][j]);
            } else {
                printf(" %c ", game->matrix[i][j]);
            }
        }
        putchar('\n');
    }
}

int check_word(GameState *game, int start_row, int start_col, int end_row, int end_col) {
    int dr = (end_row > start_row) ? 1 : (end_row < start_row) ? -1 : 0;
    int dc = (end_col > start_col) ? 1 : (end_col < start_col) ? -1 : 0;
    
    int len = (dr != 0) ? abs(end_row - start_row) + 1 : abs(end_col - start_col) + 1;
    
    char extracted[MAX_WORD_LEN + 1] = {0};
    for(int i = 0; i < len; i++) {
        extracted[i] = tolower(game->matrix[start_row + dr * i][start_col + dc * i]);
    }
    
    for(int i = 0; i < game->word_count; i++) {
        if(!game->words[i].found && strcmp(extracted, game->words[i].text) == 0) {
            game->words[i].found = true;
            game->found_words++;
            
            for(int j = 0; j < len; j++) {
                int r = start_row + dr * j;
                int c = start_col + dc * j;
                game->matrix[r][c] = '0' + (i + 1);
            }
            
            return 1;
        }
    }
    
    return 0;
}

void calculate_score(GameState *game) {
    int base_score = game->word_count * 100;
    int efficiency = (int)((float)game->found_words / game->attempts * 100);
    game->score = base_score + efficiency;
    
    print_header("RESULTADO FINAL");
    printf("Palavras encontradas: %d/%d\n", game->found_words, game->word_count);
    printf("Tentativas: %d\n", game->attempts);
    printf("Pontuação: %d\n", game->score);
    
    if(game->found_words == game->word_count) {
        printf(COLOR_GREEN "Parabéns! Você encontrou todas as palavras!\n" COLOR_RESET);
    }
    
    save_high_score(game);
}

void save_high_score(const GameState *game) {
    HighScore new_score;
    printf("Digite seu nome: ");
    fgets(new_score.player_name, MAX_PLAYER_NAME, stdin);
    new_score.player_name[strcspn(new_score.player_name, "\n")] = '\0';
    new_score.score = game->score;
    new_score.date = time(NULL);
    
    HighScore scores[MAX_HIGH_SCORES + 1];
    int count = 0;
    
    FILE *file = fopen(SCORE_FILE, "rb");
    if(file) {
        count = fread(scores, sizeof(HighScore), MAX_HIGH_SCORES, file);
        fclose(file);
    }
    
    if(count < MAX_HIGH_SCORES) {
        scores[count++] = new_score;
    } else {
        scores[MAX_HIGH_SCORES] = new_score;
        count = MAX_HIGH_SCORES + 1;
    }
    
    for(int i = 0; i < count - 1; i++) {
        for(int j = i + 1; j < count; j++) {
            if(scores[i].score < scores[j].score) {
                HighScore temp = scores[i];
                scores[i] = scores[j];
                scores[j] = temp;
            }
        }
    }
    
    file = fopen(SCORE_FILE, "wb");
    if(file) {
        fwrite(scores, sizeof(HighScore), count > MAX_HIGH_SCORES ? MAX_HIGH_SCORES : count, file);
        fclose(file);
    }
}

void show_high_scores() {
    print_header("RECORDES");
    
    HighScore scores[MAX_HIGH_SCORES];
    int count = 0;
    
    FILE *file = fopen(SCORE_FILE, "rb");
    if(file) {
        count = fread(scores, sizeof(HighScore), MAX_HIGH_SCORES, file);
        fclose(file);
    }
    
    if(count == 0) {
        printf("Nenhum recorde ainda.\n");
        return;
    }
    
    printf("%-20s %-10s %s\n", "Jogador", "Pontuação", "Data");
    for(int i = 0; i < count; i++) {
        char date_str[20];
        strftime(date_str, sizeof(date_str), "%d/%m/%Y", localtime(&scores[i].date));
        printf("%-20s %-10d %s\n", scores[i].player_name, scores[i].score, date_str);
    }
}

void show_word_manager() {
    int choice;
    do {
        print_header("GERENCIAR PALAVRAS");
        printf("1. Adicionar palavra\n");
        printf("2. Editar palavra\n");
        printf("3. Remover palavra\n");
        printf("4. Listar palavras\n");
        printf("0. Voltar\n");
        
        choice = get_valid_int("Escolha uma opção", 0, 4);
        
        switch(choice) {
            case 1: add_word_to_dict(); break;
            case 2: edit_word_in_dict(); break;
            case 3: remove_word_from_dict(); break;
            case 4: list_words_in_dict(); break;
        }
    } while(choice != 0);
}

void add_word_to_dict() {
    Word new_word;
    printf("Digite a nova palavra (mínimo %d letras): ", MIN_WORD_LEN);
    fgets(new_word.text, MAX_WORD_LEN, stdin);
    new_word.text[strcspn(new_word.text, "\n")] = '\0';
    
    if(strlen(new_word.text) < MIN_WORD_LEN) {
        printf("Palavra muito curta!\n");
        return;
    }
    
    for(int i = 0; new_word.text[i]; i++) {
        new_word.text[i] = tolower(new_word.text[i]);
    }
    
    Word *words = NULL;
    int count = load_words(WORD_FILE, &words);
    
    for(int i = 0; i < count; i++) {
        if(strcmp(words[i].text, new_word.text) == 0) {
            printf("Esta palavra já existe no dicionário!\n");
            free(words);
            return;
        }
    }
    
    Word *new_words = realloc(words, (count + 1) * sizeof(Word));
    if(!new_words) {
        printf("Erro ao alocar memória!\n");
        free(words);
        return;
    }
    
    words = new_words;
    strcpy(words[count].text, new_word.text);
    words[count].length = strlen(new_word.text);
    words[count].difficulty = 2;
    
    if(save_words(WORD_FILE, words, count + 1)) {
        printf("Palavra adicionada com sucesso!\n");
    } else {
        printf("Erro ao salvar palavra!\n");
    }
    
    free(words);
}

void edit_word_in_dict() {
    Word *words = NULL;
    int count = load_words(WORD_FILE, &words);
    if(count == 0) {
        printf("Nenhuma palavra cadastrada.\n");
        return;
    }
    
    printf("\nLista de palavras:\n");
    for(int i = 0; i < count; i++) {
        printf("%d. %s\n", i+1, words[i].text);
    }
    
    int index = get_valid_int("Digite o número da palavra a editar", 1, count) - 1;
    
    printf("Palavra atual: %s\n", words[index].text);
    printf("Nova palavra (deixe em branco para cancelar): ");
    
    char new_word[MAX_WORD_LEN];
    fgets(new_word, MAX_WORD_LEN, stdin);
    new_word[strcspn(new_word, "\n")] = '\0';
    
    if(strlen(new_word) == 0) {
        printf("Edição cancelada.\n");
        free(words);
        return;
    }
    
    if(strlen(new_word) < MIN_WORD_LEN) {
        printf("Palavra muito curta!\n");
        free(words);
        return;
    }
    
    for(int i = 0; i < count; i++) {
        if(i != index && strcmp(words[i].text, new_word) == 0) {
            printf("Esta palavra já existe no dicionário!\n");
            free(words);
            return;
        }
    }
    
    strcpy(words[index].text, new_word);
    words[index].length = strlen(new_word);
    
    if(save_words(WORD_FILE, words, count)) {
        printf("Palavra atualizada com sucesso!\n");
    } else {
        printf("Erro ao salvar alterações!\n");
    }
    
    free(words);
}

void remove_word_from_dict() {
    Word *words = NULL;
    int count = load_words(WORD_FILE, &words);
    if(count == 0) {
        printf("Nenhuma palavra cadastrada.\n");
        return;
    }
    
    printf("\nLista de palavras:\n");
    for(int i = 0; i < count; i++) {
        printf("%d. %s\n", i+1, words[i].text);
    }
    
    int index = get_valid_int("Digite o número da palavra a remover", 1, count) - 1;
    
    printf("Remover a palavra '%s'? (s/n): ", words[index].text);
    char confirm;
    scanf(" %c", &confirm);
    clear_input_buffer();
    
    if(tolower(confirm) == 's') {
        for(int i = index; i < count-1; i++) {
            words[i] = words[i+1];
        }
        
        if(save_words(WORD_FILE, words, count-1)) {
            printf("Palavra removida com sucesso!\n");
        } else {
            printf("Erro ao salvar alterações!\n");
        }
    } else {
        printf("Remoção cancelada.\n");
    }
    
    free(words);
}

void list_words_in_dict() {
    Word *words = NULL;
    int count = load_words(WORD_FILE, &words);
    if(count == 0) {
        printf("Nenhuma palavra cadastrada.\n");
        return;
    }
    
    print_header("LISTA DE PALAVRAS");
    printf("Total: %d palavras\n\n", count);
    
    for(int i = 0; i < count; i++) {
        printf("%3d. %-20s (Tamanho: %2d, Dificuldade: %d)\n", 
               i+1, words[i].text, words[i].length, words[i].difficulty);
    }
    
    free(words);
}

void show_settings() {
    print_header("CONFIGURAÇÕES");
    printf("1. Alterar arquivo de palavras\n");
    printf("2. Alterar arquivo de recordes\n");
    printf("3. Redefinir configurações\n");
    printf("0. Voltar\n");
    
    int choice = get_valid_int("Escolha uma opção", 0, 3);
    
    switch(choice) {
        case 1:
            printf("Novo caminho para arquivo de palavras: ");
            char new_word_file[256];
            fgets(new_word_file, sizeof(new_word_file), stdin);
            new_word_file[strcspn(new_word_file, "\n")] = '\0';
            WORD_FILE = strdup(new_word_file);
            break;
        case 2:
            printf("Novo caminho para arquivo de recordes: ");
            char new_score_file[256];
            fgets(new_score_file, sizeof(new_score_file), stdin);
            new_score_file[strcspn(new_score_file, "\n")] = '\0';
            SCORE_FILE = strdup(new_score_file);
            break;
        case 3:
            WORD_FILE = "palavras.dat";
            SCORE_FILE = "scores.dat";
            printf("Configurações redefinidas para padrão.\n");
            break;
    }
}

int get_valid_int(const char *prompt, int min, int max) {
    int value;
    while(1) {
        printf("%s (%d-%d): ", prompt, min, max);
        if(scanf("%d", &value) != 1) {
            printf("Entrada inválida. Digite um número.\n");
            clear_input_buffer();
            continue;
        }
        if(value >= min && value <= max) {
            clear_input_buffer();
            return value;
        }
        printf("Valor deve estar entre %d e %d.\n", min, max);
    }
}

void clear_input_buffer() {
    int c;
    while((c = getchar()) != '\n' && c != EOF);
}

void print_centered(const char *text) {
    int width = 50;
    int len = strlen(text);
    int padding = (width - len) / 2;
    
    for(int i = 0; i < padding; i++) putchar(' ');
    printf("%s\n", text);
}

void print_header(const char *title) {
    printf("\n");
    for(int i = 0; i < 50; i++) putchar('=');
    printf("\n");
    print_centered(title);
    for(int i = 0; i < 50; i++) putchar('=');
    printf("\n");
}
