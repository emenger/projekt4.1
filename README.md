# projekt4.1

main.cpp:

#include <iostream>
#include <memory>
#include "maze.h"
#include "player.h"
#include "game_state.h"
#include "ghost_base.h"
#include "ghost_factory.h"
#include "steps_to_goal.h"
#include "std_lib_inc (2).h"


// ---------------------------------------------------------------------------
// Hilfsfunktionen
// ---------------------------------------------------------------------------

// Bewegt die Spielerfigur entsprechend der Eingabe ('w', 'a', 's', 'd')
void movePlayer(GameState& state, char direction) {
    // Aktuelle Position ermitteln
    int r = state.getPlayer().getPositionRow();
    int c = state.getPlayer().getPositionCol();

    // Neue Zielkoordinaten
    int newR = r;
    int newC = c;

    // Richtung auswerten
    switch (direction) {
        case 'w': // Aufwärts
            newR--;
            break;
        case 's': // Abwärts
            newR++;
            break;
        case 'a': // Links
            newC--;
            break;
        case 'd': // Rechts
            newC++;
            break;
        default:
            // Unerwartetes Kommando => keine Bewegung
            return;
    }

    // Zelleninhalt prüfen
    const Maze& maze = state.getMaze();
    if (newR < 0 || newR >= maze.getRows() ||
        newC < 0 || newC >= maze.getCols()) {
        // Außerhalb des Labyrinths -> Bewegung ignorieren
        return;
    }

    char cell = maze.getCell(newR, newC);
    // Hier kann man definieren, welche Felder begehbar sind.
    // Bsp.: '.', 'K', 'Z', 'A' -> begehbar
    // 'T' (Tür) ist nur begehbar, wenn man Schlüssel hat, usw.
    // Für den Einstieg behandeln wir '#' und 'T' als Hindernisse.
    if (cell == '#' || cell == 'T') {
        // Bewegung blockiert
        return;
    }

    // Bewegung durchführen
    state.getPlayer().setPosition(newR, newC);
}

// Prüft, ob die Spielerfigur auf einem Geist-Feld steht
bool checkGhostCollision(GameState& state) {
    int pRow = state.getPlayer().getPositionRow();
    int pCol = state.getPlayer().getPositionCol();

    for (auto& ghost : state.getGhosts()) {
        if (ghost->getRow() == pRow && ghost->getCol() == pCol) {
            return true;
        }
    }
    return false;
}

// Gibt das Labyrinth zusammen mit der Spielerposition und ggf. den Entfernungsinformationen aus
void displayMaze(const GameState& state) {
    const Maze& maze = state.getMaze();

    // Ziel in diesem Beispiel fix angenommen (z.B. [5,5]),
    // falls dein Code das Ziel woanders speichert, bitte anpassen
    int goalRow = 5;
    int goalCol = 5;

    // Im Info-Modus werden die Schritte bis zum Ziel berechnet (max. 5)
    if (state.isInfoMode()) {
        int steps = computeStepsToGoal(maze,
                                       state.getPlayer().getPositionRow(),
                                       state.getPlayer().getPositionCol(),
                                       5,
                                       goalRow, goalCol);
        if (steps <= 5) {
            cout << "(Schritte bis zum Ziel: " << steps << ")\n";
        }
    }

    // Labyrinth ausgeben
    int totalRows = maze.getRows();
    int totalCols = maze.getCols();

    for (int r = 0; r < totalRows; r++) {
        for (int c = 0; c < totalCols; c++) {
            // Prüfen, ob hier die Spielerfigur ist
            if (r == state.getPlayer().getPositionRow() &&
                c == state.getPlayer().getPositionCol()) {
                cout << 'S' << " ";
            } else {
                // Normales Labyrinth-Zeichen
                cout << maze.getCell(r, c) << " ";
            }
        }
        cout << "\n";
    }
}

// Gibt eine kurze Hilfe zu den Befehlen aus
void displayHelp() {
    cout << "Willkommen im Labyrinth!\n"
              << "Die moeglichen Befehle sind:\n"
              << "  w, a, s, d - Bewegung\n"
              << "  i - Informationsmodus umschalten\n"
              << "  h - Hilfe anzeigen\n"
              << "  q - Spiel beenden\n";
}

// Initialisiert Beispiel-Labyrinth mit Geistern
GameState initializeGame() {
    // Beispiel-Größe des Labyrinths
    int rows = 7;
    int cols = 7;
    vector<vector<char>> data(rows, vector<char>(cols, '.'));

    // Beispiel: Wände, Türen, Ziel
    data[1][2] = 'T';  // Tür
    data[2][3] = '#';  // Wand
    data[5][5] = 'Z';  // Ziel
    data[3][3] = 'C';  // Connelly-Geist
    data[4][1] = 'B';  // Bowie-Geist

    // Labyrinth-Objekt anlegen
    Maze maze(rows, cols, data);

    // Spielerfigur am Start (0,0)
    Player player(0, 0);

    // GameState erstellen
    GameState state(maze, player);

    // Geister finden und im State ablegen
    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < cols; c++) {
            char cell = data[r][c];
            if (cell == 'A' || cell == 'B' || cell == 'C') {
                // Geist erzeugen
                state.addGhost(std::unique_ptr<GhostBase>(createGhost(r, c, cell)));
                // Für die Anzeige als 'A' markieren
                state.getMaze().setCell(r, c, 'A');
            }
        }
    }
    return state;
}

// ---------------------------------------------------------------------------
// Hauptfunktion
// ---------------------------------------------------------------------------
int main() {
    try {
        // Spielzustand initialisieren
        GameState state = initializeGame();

        // Hauptschleife
        while (!state.getExit()) {
            // Aktuelles Labyrinth anzeigen
            displayMaze(state);

            // Prüfen, ob direkt zu Spielbeginn eventuell Kollisionen existieren
            if (checkGhostCollision(state)) {
                cout << "Sie haben einen Geist getroffen! Game Over!\n";
                break;
            }

            // Befehl lesen
            char cmd;
            cin >> cmd;
            if (!cin) {
                // Eingabe ungültig oder EOF -> raus aus Schleife
                break;
            }

            // Auswertung des Befehls
            switch (cmd) {
                case 'w':
                case 'a':
                case 's':
                case 'd':
                    // Spieler bewegen
                    movePlayer(state, cmd);

                    // Erneut auf Kollision prüfen
                    if (checkGhostCollision(state)) {
                        cout << "Sie haben einen Geist getroffen! Game Over!\n";
                        state.setExit(true);
                        break;
                    }

                    // Geister bewegen
                    for (auto& ghost : state.getGhosts()) {
                        ghost->move(state);
                    }

                    // Kollisionsprüfung nach Geisterbewegung
                    if (checkGhostCollision(state)) {
                        cout << "Sie haben einen Geist getroffen! Game Over!\n";
                        state.setExit(true);
                    }
                    break;

                case 'i':
                case 'I':
                    // Info-Modus umschalten
                    state.toggleInfoMode();
                    break;

                case 'h':
                case 'H':
                    displayHelp();
                    break;

                case 'q':
                case 'Q':
                    state.setExit(true);
                    break;

                default:
                    cout << "Unbekannter Befehl: " << cmd << "\n";
                    break;
            }
        }

        cout << "Spiel wird beendet. Schoenen Tag noch!\n";
        return 0;
    }
    catch (const exception& e) {
        cerr << "Fehler: " << e.what() << endl;
        return 1;
    }
}



game_state.cpp:

#include "game_state.h"



game_state.h:

#include <vector>
#include <memory>
#include "maze.h"
#include "player.h"
#include "ghost_base.h"

#ifndef GAME_STATE_H
#define GAME_STATE_H

class GhostBase; // Vorwärtsdeklaration

// Hält den kompletten Spielzustand
class GameState {
private:
    Maze maze_;
    Player player_;
    bool exit_ = false;     // Ob das Spiel beendet werden soll
    bool infoMode_ = false; // Ob der Informations-Modus aktiv ist

    // Container für alle Geister
    vector<unique_ptr<GhostBase>> ghosts_;

public:
    GameState(const Maze& maze, const Player& player)
        : maze_(maze), player_(player) {}

    Maze& getMaze() { return maze_; }
    const Maze& getMaze() const { return maze_; }

    Player& getPlayer() { return player_; }
    const Player& getPlayer() const { return player_; }

    void setExit(bool val) { exit_ = val; }
    bool getExit() const { return exit_; }

    void toggleInfoMode() { infoMode_ = !infoMode_; }
    bool isInfoMode() const { return infoMode_; }

    // Methoden zum Verwalten der Geister
    void addGhost(unique_ptr<GhostBase> g) {
        ghosts_.push_back(std::move(g));
    }

    vector<unique_ptr<GhostBase>>& getGhosts() {
        return ghosts_;
    }
};

#endif //GAME_STATE_H



ghost_base.cpp:

#include "ghost_base.h"


ghost_base.h:

#pragma once
#include <vector>


#ifndef GHOST_BASE_H
#define GHOST_BASE_H

// Vorwärtsdeklaration, um Kreisabhängigkeiten zu vermeiden
class GameState;

// Aufzählungstyp für die verschiedenen Geist-Arten
enum class GhostType {
    ANIMATRONIC,  // Bleibt an derselben Stelle
    BOWIE,        // Bewegt sich horizontal (links-rechts)
    CONNELLY      // Versucht, die SpielerIn zu verfolgen
};

// Basisklasse für alle Geister
class GhostBase {
protected:
    int row_;
    int col_;
    GhostType type_;

public:
    GhostBase(int row, int col, GhostType type)
        : row_(row), col_(col), type_(type) {}

    virtual ~GhostBase() = default;

    // Getter-Methoden
    int getRow() const { return row_; }
    int getCol() const { return col_; }
    GhostType getType() const { return type_; }

    // Wird nach jeder erfolgreichen Bewegung der SpielerIn aufgerufen,
    // damit der Geist sich aktualisieren kann.
    virtual void move(GameState& state) = 0;
};


#endif //GHOST_BASE_H


ghost_factory.cpp

#include "ghost_factory.h"


ghost_factory.h:

#pragma once


#ifndef GHOST_FACTORY_H
#define GHOST_FACTORY_H

// Vorwärtsdeklaration der Fabrikfunktion, damit andere Dateien sie nutzen können
class GhostBase;

GhostBase* createGhost(int row, int col, char symbol);



#endif //GHOST_FACTORY_H


ghosts.cpp:

#include "ghost_base.h"
#include "game_state.h"  // Für Zugriff auf Maze & Player & andere Definitionen
#include <cmath>         // Für std::abs


// Animatronics-Geist: Bleibt an seiner Position
class GhostAnimatronic : public GhostBase {
public:
    GhostAnimatronic(int row, int col)
        : GhostBase(row, col, GhostType::ANIMATRONIC) {}

    void move(GameState& /*state*/) override
    {
        // Keine Bewegung
    }
};

// Bowie-Geist: bewegt sich zunächst nach links, bis er nicht weiterkommt.
// Dann wechselt er die Richtung. Wenn beide Richtungen blockiert sind, bleibt er stehen.
class GhostBowie : public GhostBase {
private:
    bool movingLeft_ = true; // Bowies starten nach links
public:
    GhostBowie(int row, int col)
        : GhostBase(row, col, GhostType::BOWIE) {}

    void move(GameState& state) override
    {
        int nextCol = movingLeft_ ? col_ - 1 : col_ + 1;
        if (canMoveTo(row_, nextCol, state)) {
            col_ = nextCol;
        } else {
            // Richtung umkehren
            movingLeft_ = !movingLeft_;
            nextCol = movingLeft_ ? col_ - 1 : col_ + 1;
            // Prüfen, ob in die andere Richtung Bewegung möglich ist
            if (canMoveTo(row_, nextCol, state)) {
                col_ = nextCol;
            }
        }
    }

private:
    // Überprüft, ob das Feld begehbar ist
    bool canMoveTo(int r, int c, const GameState& state) const
    {
        if (r < 0 || r >= state.getMaze().getRows() ||
            c < 0 || c >= state.getMaze().getCols()) {
            return false;
        }

        char cell = state.getMaze().getCell(r, c);
        // Bowie-Geister behandeln '#' und 'T' als Hindernisse
        if (cell == '#' || cell == 'T') {
            return false;
        }
        return true;
    }
};

// Connelly-Geist: versucht aktiv, die SpielerIn zu fangen,
// indem er den größeren Abstand (Zeile oder Spalte) zuerst verringert.
class GhostConnelly : public GhostBase {
public:
    GhostConnelly(int row, int col)
        : GhostBase(row, col, GhostType::CONNELLY) {}

    void move(GameState& state) override
    {
        int playerRow = state.getPlayer().getPositionRow();
        int playerCol = state.getPlayer().getPositionCol();

        int dRow = playerRow - row_;
        int dCol = playerCol - col_;
        int absRow = abs(dRow);
        int absCol = abs(dCol);

        // Bestimmen, ob wir uns zuerst vertikal (Zeile) oder horizontal (Spalte) bewegen
        bool preferVertical = (absRow >= absCol);

        // Erster Versuch in der bevorzugten Richtung
        if (!moveInPreferredDirection(preferVertical, dRow, dCol, state)) {
            // Zweiter Versuch in der anderen Richtung
            moveInPreferredDirection(!preferVertical, dRow, dCol, state);
        }
    }

private:
    bool moveInPreferredDirection(bool vertical, int dRow, int dCol, GameState& state)
    {
        int nextRow = row_;
        int nextCol = col_;

        if (vertical) {
            // Einen Schritt in Richtung dRow
            if (dRow > 0)      nextRow = row_ + 1;
            else if (dRow < 0) nextRow = row_ - 1;
        } else {
            // Einen Schritt in Richtung dCol
            if (dCol > 0)      nextCol = col_ + 1;
            else if (dCol < 0) nextCol = col_ - 1;
        }

        if (canMoveTo(nextRow, nextCol, state)) {
            row_ = nextRow;
            col_ = nextCol;
            return true;
        }
        return false;
    }

    bool canMoveTo(int r, int c, const GameState& state) const
    {
        if (r < 0 || r >= state.getMaze().getRows() ||
            c < 0 || c >= state.getMaze().getCols()) {
            return false;
        }

        char cell = state.getMaze().getCell(r, c);
        // Connelly-Geister behandeln '#' und 'T' als unüberwindbare Hindernisse
        if (cell == '#' || cell == 'T') {
            return false;
        }
        return true;
    }
};

// Fabrikfunktion, die basierend auf einem Symbol den passenden Geist erzeugt
GhostBase* createGhost(int row, int col, char symbol)
{
    switch (symbol) {
    case 'A': // Animatronic
        return new GhostAnimatronic(row, col);
    case 'B': // Bowie
        return new GhostBowie(row, col);
    case 'C': // Connelly
        return new GhostConnelly(row, col);
    default:
        return nullptr; // Oder eine Exception auslösen
    }
}


ghosts.h:

#ifndef GHOSTS_H
#define GHOSTS_H



class ghosts {

};



#endif //GHOSTS_H


maze.cpp:

#include "maze.h"


maze.h:

#pragma once
#include <vector>
#include <string>
using namespace std;


#ifndef MAZE_H
#define MAZE_H

class Maze {
private:
    int rows_;
    int cols_;
    vector<vector<char>> data_; // Enthält die Labyrinth-Daten

public:
    Maze(int rows, int cols, const vector<vector<char>>& data)
        : rows_(rows), cols_(cols), data_(data) {}

    int getRows() const { return rows_; }
    int getCols() const { return cols_; }

    // “Getter” für den Inhalt einer Zelle
    char getCell(int r, int c) const {
        return data_[r][c];
    }

    // “Setter” für den Inhalt einer Zelle
    void setCell(int r, int c, char ch) {
        data_[r][c] = ch;
    }
};


player.cpp:

#include "player.h"


player.h:

#pragma once
#include <vector>



#ifndef PLAYER_H
#define PLAYER_H

class Player {
private:
    int noKeys_;    // Anzahl an Schlüsseln
    int row_, col_; // Position der SpielerIn

public:
    Player(int row, int col)
        : noKeys_(0), row_(row), col_(col) {}

    int getKeys() const { return noKeys_; }
    void addKey() { noKeys_++; }
    void useKey() { if (noKeys_ > 0) noKeys_--; }

    int getPositionRow() const { return row_; }
    int getPositionCol() const { return col_; }

    void setPosition(int r, int c) {
        row_ = r;
        col_ = c;
    }
};

#endif //PLAYER_H


steps_to_goal.cpp:

#include "steps_to_goal.h"


// Rekursive Berechnung der kleinsten Schrittanzahl zum Ziel. Gibt INF_STEPS zurück, falls unerreichbar.
int computeStepsToGoal(const Maze& maze, int row, int col, int maxDepth,
                       int goalRow, int goalCol)
{
    // Position außerhalb des Labyrinths oder nicht begehbar => INF
    if (row < 0 || row >= maze.getRows() ||
        col < 0 || col >= maze.getCols()) {
        return INF_STEPS;
        }

    char cell = maze.getCell(row, col);
    // '#' oder 'T' behandeln wir als unpassierbar für diese Entfernungsberechnung
    if (cell == '#' || cell == 'T') {
        return INF_STEPS;
    }

    // Wenn wir das Ziel erreicht haben
    if (row == goalRow && col == goalCol) {
        return 0;
    }

    // Wenn die Suchtiefe aufgebraucht ist
    if (maxDepth == 0) {
        return INF_STEPS;
    }

    // Rekursiv die Nachbarn untersuchen (oben, unten, links, rechts)
    int oben    = computeStepsToGoal(maze, row - 1, col, maxDepth - 1, goalRow, goalCol);
    int unten   = computeStepsToGoal(maze, row + 1, col, maxDepth - 1, goalRow, goalCol);
    int links   = computeStepsToGoal(maze, row, col - 1, maxDepth - 1, goalRow, goalCol);
    int rechts  = computeStepsToGoal(maze, row, col + 1, maxDepth - 1, goalRow, goalCol);

    int minSteps = oben;
    if (unten < minSteps)   minSteps = unten;
    if (links < minSteps)   minSteps = links;
    if (rechts < minSteps)  minSteps = rechts;

    if (minSteps == INF_STEPS) {
        return INF_STEPS;
    }
    return 1 + minSteps;
}


steps_to_goal.h:

#pragma once
#include "maze.h"


#ifndef STEPS_TO_GOAL_H
#define STEPS_TO_GOAL_H

// Konstante, um "unendlich viele Schritte" zu repräsentieren
constexpr int INF_STEPS = 99999;

// Rekursive Funktion zur Berechnung der Schritte zum Ziel (maximal 5 Tiefe)
int computeStepsToGoal(const Maze& maze, int row, int col, int maxDepth,
                       int goalRow, int goalCol);



#endif //STEPS_TO_GOAL_H
