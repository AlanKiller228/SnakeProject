# SnakeProject
A classic Snake game built using clean and structured code. The player controls a snake that moves around the screen, eats food, and grows longer. The goal is to score as many points as possible without colliding with the walls or the snake’s own body.
#include <SFML/Graphics.hpp>
#include <vector>
#include <cstdlib>
#include <ctime>
#include <cmath>
#include <algorithm>
#include <string>
#include <fstream>
#include <sstream>
#include <iomanip>

struct Vec2i {
    int x, y;
    bool operator==(const Vec2i& o) const { return x == o.x && y == o.y; }
};

enum Dir { Up, Down, Left, Right };

static sf::Vector2f lerp(const sf::Vector2f& a, const sf::Vector2f& b, float t) { return a + (b - a) * t; }
static float vlen(const sf::Vector2f& v) { return std::sqrt(v.x * v.x + v.y * v.y); }
static float radToDeg(float r) { return r * 180.f / 3.14159265f; }

static bool isOpposite(Dir a, Dir b) {
    return (a == Up && b == Down) || (a == Down && b == Up) ||
           (a == Left && b == Right) || (a == Right && b == Left);
}

// ===================== TOP-10 SCORES =====================
struct ScoreEntry {
    std::string name;
    int score = 0;
    std::string date; // YYYY-MM-DD
};

static std::string todayISO() {
    std::time_t t = std::time(nullptr);
    std::tm* now = std::localtime(&t);
    char buf[32];
    std::strftime(buf, sizeof(buf), "%Y-%m-%d", now);
    return buf;
}

static std::vector<ScoreEntry> loadScores(const std::string& path) {
    std::vector<ScoreEntry> list;
    std::ifstream f(path);
    if (!f) return list;

    // format: NAME SCORE YYYY-MM-DD
    ScoreEntry e;
    while (f >> e.name >> e.score >> e.date) {
        if (e.score < 0) continue;
        list.push_back(e);
    }

    std::sort(list.begin(), list.end(), [](const ScoreEntry& a, const ScoreEntry& b) {
        return a.score > b.score;
    });

    if (list.size() > 10) list.resize(10);
    return list;
}

static void saveScores(const std::string& path, const std::vector<ScoreEntry>& list) {
    std::ofstream f(path, std::ios::trunc);
    if (!f) return;
    for (const auto& e : list) {
        f << e.name << " " << e.score << " " << e.date << "\n";
    }
}

static void addScore(std::vector<ScoreEntry>& list, const std::string& name, int score) {
    if (score < 0) return;

    ScoreEntry e;
    e.name = name.empty() ? "PLAYER" : name;
    e.score = score;
    e.date = todayISO();

    list.push_back(e);
    std::sort(list.begin(), list.end(), [](const ScoreEntry& a, const ScoreEntry& b) {
        return a.score > b.score;
    });

    if (list.size() > 10) list.resize(10);
}

// ===================== SAVEGAME (CONTINUE) =====================
// Simple text save. If file exists and is valid -> load.
static bool saveGame(const std::string& path,
                     const std::vector<Vec2i>& snake,
                     const std::vector<Vec2i>& apples,
                     int score,
                     float stepTime,
                     Dir moveDir,
                     Dir queuedDir)
{
    std::ofstream f(path, std::ios::trunc);
    if (!f) return false;

    // version header
    f << "SNAKE_SAVE_V1\n";
    f << score << "\n";
    f << std::fixed << std::setprecision(6) << stepTime << "\n";
    f << (int)moveDir << " " << (int)queuedDir << "\n";

    f << snake.size() << "\n";
    for (auto& s : snake) f << s.x << " " << s.y << "\n";

    f << apples.size() << "\n";
    for (auto& a : apples) f << a.x << " " << a.y << "\n";

    return true;
}

static bool loadGame(const std::string& path,
                     std::vector<Vec2i>& snake,
                     std::vector<Vec2i>& apples,
                     int& score,
                     float& stepTime,
                     Dir& moveDir,
                     Dir& queuedDir)
{
    std::ifstream f(path);
    if (!f) return false;

    std::string header;
    std::getline(f, header);
    if (header != "SNAKE_SAVE_V1") return false;

    int md = 0, qd = 0;
    size_t nSnake = 0, nApples = 0;

    if (!(f >> score)) return false;
    if (!(f >> stepTime)) return false;
    if (!(f >> md >> qd)) return false;

    if (md < 0 || md > 3 || qd < 0 || qd > 3) return false;
    moveDir = (Dir)md;
    queuedDir = (Dir)qd;

    if (!(f >> nSnake)) return false;
    if (nSnake < 2 || nSnake > 2000) return false;

    snake.clear();
    for (size_t i = 0; i < nSnake; i++) {
        Vec2i v{};
        if (!(f >> v.x >> v.y)) return false;
        snake.push_back(v);
    }

    if (!(f >> nApples)) return false;
    if (nApples > 50) return false;

    apples.clear();
    for (size_t i = 0; i < nApples; i++) {
        Vec2i a{};
        if (!(f >> a.x >> a.y)) return false;
        apples.push_back(a);
    }

    if (score < 0) score = 0;
    if (stepTime < 0.02f) stepTime = 0.02f;
    if (stepTime > 1.0f) stepTime = 1.0f;

    return true;
}

int main() {
    std::srand((unsigned)std::time(nullptr));

    const std::string SCORES_PATH = "scores.txt";
    const std::string SAVE_PATH   = "savegame.txt";
    const std::string PLAYER_NAME = "PLAYER"; // you can change later to ask user

    // Load top10 scores
    std::vector<ScoreEntry> scores = loadScores(SCORES_PATH);
    int bestScore = scores.empty() ? 0 : scores.front().score;

    const int cellSize = 24;
    const int gridW = 25;
    const int gridH = 25;
    const int winW = gridW * cellSize;
    const int winH = gridH * cellSize;

    const sf::Color grassA(170, 215, 81);
    const sf::Color grassB(162, 209, 73);

    const sf::Color headColor(56, 142, 60);
    const sf::Color bodyColor(76, 175, 80);

    sf::RenderWindow window(sf::VideoMode(winW, winH), "Snake");
    window.setFramerateLimit(60);

    sf::Font font;
    bool hasFont = false;
    if (font.loadFromFile("assets/fonts/Baloo2-Bold.ttf")) hasFont = true;
    else if (font.loadFromFile("/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf")) hasFont = true;

    std::vector<Vec2i> snake, prevSnake;
    std::vector<Vec2i> apples;

    Dir moveDir = Right;    // last performed move
    Dir queuedDir = Right;  // queued for next step

    bool started = false, paused = false, gameOver = false;
    bool savedThisGameOver = false;

    int score = 0;

    float stepTime = 0.13f;
    float timer = 0.f;
    sf::Clock dtClock;
    sf::Clock tClock;

    auto inBounds = [&](const Vec2i& p) { return p.x >= 0 && p.x < gridW && p.y >= 0 && p.y < gridH; };

    auto cellCenter = [&](const Vec2i& c) -> sf::Vector2f {
        return { c.x * (float)cellSize + cellSize * 0.5f, c.y * (float)cellSize + cellSize * 0.5f };
    };

    auto cellOccupiedBySnake = [&](const Vec2i& p) {
        for (auto& s : snake) if (s == p) return true;
        return false;
    };

    auto cellOccupiedByApples = [&](const Vec2i& p) {
        for (auto& a : apples) if (a == p) return true;
        return false;
    };

    auto targetAppleCount = [&](int sc) -> int {
        if (sc >= 200) return 4;
        if (sc >= 120) return 3;
        if (sc >= 50)  return 2;
        return 1;
    };

    auto spawnOneApple = [&]() -> Vec2i {
        while (true) {
            Vec2i p{ std::rand() % gridW, std::rand() % gridH };
            if (!cellOccupiedBySnake(p) && !cellOccupiedByApples(p)) return p;
        }
    };

    auto syncApplesToScore = [&]() {
        int need = targetAppleCount(score);
        while ((int)apples.size() < need) apples.push_back(spawnOneApple());
        while ((int)apples.size() > need) apples.pop_back();
    };

    auto resetGame = [&]() {
        snake.clear();
        snake.push_back({ gridW / 2, gridH / 2 });
        snake.push_back({ gridW / 2 - 1, gridH / 2 });
        snake.push_back({ gridW / 2 - 2, gridH / 2 });
        prevSnake = snake;

        moveDir = Right;
        queuedDir = Right;

        started = false;
        paused = false;
        gameOver = false;
        savedThisGameOver = false;

        score = 0;
        stepTime = 0.12f;
        timer = 0.f;

        apples.clear();
        syncApplesToScore();

        // start a fresh save immediately
        saveGame(SAVE_PATH, snake, apples, score, stepTime, moveDir, queuedDir);
    };

    // Graphics shapes
    sf::RectangleShape tile(sf::Vector2f((float)cellSize, (float)cellSize));
    tile.setOutlineThickness(0.f);

    const float pad = 3.f;
    const float bodyR = (cellSize * 0.5f) - pad;
    const float headR = bodyR + 1.f;

    sf::CircleShape endCap(bodyR);
    endCap.setPointCount(40);
    endCap.setFillColor(bodyColor);

    sf::RectangleShape linkRect;
    linkRect.setFillColor(bodyColor);

    sf::CircleShape headCircle(headR);
    headCircle.setPointCount(50);
    headCircle.setFillColor(headColor);

    sf::CircleShape eyeWhite(cellSize * 0.12f);
    eyeWhite.setFillColor(sf::Color::White);
    eyeWhite.setPointCount(30);

    sf::CircleShape pupil(cellSize * 0.06f);
    pupil.setFillColor(sf::Color::Black);
    pupil.setPointCount(30);

    auto setCircleCenter = [&](sf::CircleShape& circ, const sf::Vector2f& center) {
        float r = circ.getRadius();
        circ.setPosition(center.x - r, center.y - r);
    };

    auto drawCapsule = [&](const sf::Vector2f& p1, const sf::Vector2f& p2, const sf::Color& color) {
        sf::Vector2f d = p2 - p1;
        float dist = vlen(d);
        if (dist < 0.001f) return;
        float angle = radToDeg(std::atan2(d.y, d.x));

        linkRect.setFillColor(color);
        linkRect.setSize({ dist, bodyR * 2.f });
        linkRect.setOrigin(0.f, bodyR);
        linkRect.setPosition(p1);
        linkRect.setRotation(angle);
        window.draw(linkRect);

        endCap.setFillColor(color);
        setCircleCenter(endCap, p1);
        window.draw(endCap);
        setCircleCenter(endCap, p2);
        window.draw(endCap);
    };

    auto drawTextShadow = [&](sf::Text txt, float x, float y) {
        txt.setPosition(x + 2.f, y + 2.f);
        txt.setFillColor(sf::Color(0, 0, 0, 170));
        window.draw(txt);
        txt.setPosition(x, y);
        txt.setFillColor(sf::Color::White);
        window.draw(txt);
    };

    sf::RectangleShape panel(sf::Vector2f((float)winW * 0.72f, (float)winH * 0.28f));
    panel.setFillColor(sf::Color(0, 0, 0, 120));
    panel.setOrigin(panel.getSize().x / 2.f, panel.getSize().y / 2.f);

    sf::Text tScore, tBest, tHint, tOver, tPause;
    if (hasFont) {
        tScore.setFont(font);
        tBest.setFont(font);
        tHint.setFont(font);
        tOver.setFont(font);
        tPause.setFont(font);

        tScore.setCharacterSize(26);
        tBest.setCharacterSize(26);
        tHint.setCharacterSize(20);
        tOver.setCharacterSize(30);
        tPause.setCharacterSize(30);
    }

    auto drawAppleGoogleStyle = [&](const Vec2i& a, float pulse, float t) {
        float ax = a.x * cellSize;
        float ay = a.y * cellSize;
        sf::Vector2f c(ax + cellSize * 0.5f, ay + cellSize * 0.56f);

        float R = cellSize * 0.34f * pulse;
        float wobble = 2.5f * std::sin(t * 2.2f + (a.x * 7 + a.y * 13) * 0.1f);

        sf::Color redMain(230, 60, 55);
        sf::Color redDark(185, 35, 35, 220);
        sf::Color outline(140, 20, 20, 240);
        sf::Color highlight(255, 230, 230, 210);

        sf::Color stemBrown(120, 75, 35, 255);
        sf::Color leafGreen(70, 190, 95, 240);
        sf::Color leafVein(35, 130, 70, 220);

        {
            sf::CircleShape shadow(R * 0.85f);
            shadow.setPointCount(50);
            shadow.setFillColor(sf::Color(0, 0, 0, 55));
            shadow.setPosition(c.x - shadow.getRadius(), c.y - shadow.getRadius() + R * 0.70f);
            shadow.setScale(1.25f, 0.55f);
            window.draw(shadow);
        }

        sf::CircleShape base(R * 1.02f);
        base.setPointCount(60);
        base.setFillColor(outline);
        base.setPosition(c.x - base.getRadius(), c.y - base.getRadius());
        base.setScale(1.10f, 0.95f);
        base.setRotation(wobble);
        window.draw(base);

        sf::CircleShape body(R);
        body.setPointCount(60);
        body.setFillColor(redMain);
        body.setPosition(c.x - body.getRadius(), c.y - body.getRadius());
        body.setScale(1.10f, 0.95f);
        body.setRotation(wobble);
        window.draw(body);

        {
            sf::CircleShape shade(R * 0.92f);
            shade.setPointCount(60);
            shade.setFillColor(redDark);
            shade.setPosition(c.x - shade.getRadius() + R * 0.18f, c.y - shade.getRadius() + R * 0.22f);
            shade.setScale(1.08f, 0.90f);
            shade.setRotation(wobble);
            window.draw(shade);
        }

        {
            sf::CircleShape hl(R * 0.50f);
            hl.setPointCount(50);
            hl.setFillColor(highlight);
            hl.setPosition(c.x - hl.getRadius() - R * 0.18f, c.y - hl.getRadius() - R * 0.22f);
            hl.setScale(0.78f, 1.15f);
            hl.setRotation(wobble - 10.f);
            window.draw(hl);

            sf::CircleShape hl2(R * 0.18f);
            hl2.setPointCount(40);
            hl2.setFillColor(sf::Color(255, 245, 245, 200));
            hl2.setPosition(c.x - hl2.getRadius() - R * 0.30f, c.y - hl2.getRadius() - R * 0.05f);
            hl2.setScale(0.9f, 1.2f);
            hl2.setRotation(wobble);
            window.draw(hl2);
        }

        {
            sf::CircleShape dimple(R * 0.28f);
            dimple.setPointCount(50);
            dimple.setFillColor(sf::Color(120, 15, 15, 140));
            dimple.setPosition(c.x - dimple.getRadius(), c.y - dimple.getRadius() - R * 0.70f);
            dimple.setScale(1.25f, 0.70f);
            dimple.setRotation(wobble);
            window.draw(dimple);
        }

        {
            sf::RectangleShape stem(sf::Vector2f(R * 0.24f, R * 0.70f));
            stem.setFillColor(stemBrown);
            stem.setOrigin(stem.getSize().x / 2.f, stem.getSize().y);
            stem.setPosition(c.x + R * 0.02f, c.y - R * 0.75f);
            stem.setRotation(-18.f + wobble * 0.4f);
            window.draw(stem);

            sf::CircleShape stemCap(R * 0.13f);
            stemCap.setPointCount(30);
            stemCap.setFillColor(sf::Color(90, 55, 22, 220));
            stemCap.setPosition((c.x + R * 0.05f) - stemCap.getRadius(), (c.y - R * 0.70f) - stemCap.getRadius());
            window.draw(stemCap);
        }

        {
            sf::ConvexShape leaf;
            leaf.setPointCount(4);
            leaf.setPoint(0, sf::Vector2f(0.f, 0.f));
            leaf.setPoint(1, sf::Vector2f(R * 0.80f, R * 0.15f));
            leaf.setPoint(2, sf::Vector2f(R * 0.55f, R * 0.75f));
            leaf.setPoint(3, sf::Vector2f(R * 0.05f, R * 0.45f));
            leaf.setFillColor(leafGreen);
            leaf.setOrigin(R * 0.12f, R * 0.28f);
            leaf.setPosition(c.x + R * 0.12f, c.y - R * 0.98f);
            leaf.setRotation(-40.f + wobble * 0.6f);
            window.draw(leaf);

            sf::RectangleShape vein(sf::Vector2f(R * 0.70f, 2.f));
            vein.setFillColor(leafVein);
            vein.setOrigin(0.f, 1.f);
            vein.setPosition(c.x + R * 0.10f, c.y - R * 1.00f);
            vein.setRotation(-40.f + wobble * 0.6f);
            window.draw(vein);
        }
    };

    // Try to load savegame (Continue)
    bool hasSave = loadGame(SAVE_PATH, snake, apples, score, stepTime, moveDir, queuedDir);

    if (hasSave) {
        // Basic validation: everything must be within bounds
        bool ok = true;
        for (auto& s : snake) if (!inBounds(s)) { ok = false; break; }
        for (auto& a : apples) if (!inBounds(a)) { ok = false; break; }
        if (!ok) {
            hasSave = false;
            resetGame();
        } else {
            prevSnake = snake;
            started = false;     // show start overlay, but it's loaded
            paused = false;
            gameOver = false;
            savedThisGameOver = false;
        }
    } else {
        resetGame();
    }

    while (window.isOpen()) {
        float dt = dtClock.restart().asSeconds();
        if (!paused) timer += dt;

        sf::Event event{};
        while (window.pollEvent(event)) {
            if (event.type == sf::Event::Closed) {
                saveGame(SAVE_PATH, snake, apples, score, stepTime, moveDir, queuedDir);
                saveScores(SCORES_PATH, scores);
                window.close();
            }

            if (event.type == sf::Event::KeyPressed) {
                if (event.key.code == sf::Keyboard::Escape) {
                    saveGame(SAVE_PATH, snake, apples, score, stepTime, moveDir, queuedDir);
                    saveScores(SCORES_PATH, scores);
                    window.close();
                }

                if (event.key.code == sf::Keyboard::R) resetGame();

                if (event.key.code == sf::Keyboard::P) {
                    if (!gameOver) paused = !paused;
                }

                if (event.key.code == sf::Keyboard::Space) {
                    if (!gameOver) { started = true; paused = false; }
                }

                auto tryQueue = [&](Dir d) {
                    // Block 180-degree reversal relative to last PERFORMED move (fix "neck flip" bug)
                    if (!isOpposite(d, moveDir)) {
                        queuedDir = d;
                        started = true;
                        paused = false;
                    }
                };

                if (!gameOver) {
                    if (event.key.code == sf::Keyboard::Up)    tryQueue(Up);
                    if (event.key.code == sf::Keyboard::Down)  tryQueue(Down);
                    if (event.key.code == sf::Keyboard::Left)  tryQueue(Left);
                    if (event.key.code == sf::Keyboard::Right) tryQueue(Right);
                }
            }
        }

        if (started && !paused && !gameOver && timer >= stepTime) {
            timer = 0.f;
            prevSnake = snake;

            // Apply queued direction only on step
            moveDir = queuedDir;

            Vec2i newHead = snake.front();
            if (moveDir == Up) newHead.y -= 1;
            if (moveDir == Down) newHead.y += 1;
            if (moveDir == Left) newHead.x -= 1;
            if (moveDir == Right) newHead.x += 1;

            if (!inBounds(newHead) || cellOccupiedBySnake(newHead)) {
                gameOver = true;
                started = false;
                paused = false;

                if (!savedThisGameOver) {
                    addScore(scores, PLAYER_NAME, score);
                    saveScores(SCORES_PATH, scores);
                    bestScore = scores.empty() ? bestScore : scores.front().score;
                    savedThisGameOver = true;
                }
            } else {
                snake.insert(snake.begin(), newHead);

                bool grew = false;
                int eatenIndex = -1;
                for (int i = 0; i < (int)apples.size(); i++) {
                    if (apples[i] == newHead) { eatenIndex = i; break; }
                }

                if (eatenIndex != -1) {
                    grew = true;
                    score += 10;
                    apples[eatenIndex] = spawnOneApple();
                    syncApplesToScore();
                } else {
                    snake.pop_back();
                }

                if (grew) prevSnake.push_back(prevSnake.back());

                // autosave after each successful step
                saveGame(SAVE_PATH, snake, apples, score, stepTime, moveDir, queuedDir);
            }
        }

        syncApplesToScore();

        float alpha = 1.f;
        if (started && !paused && !gameOver) alpha = std::clamp(timer / stepTime, 0.f, 1.f);

        std::vector<sf::Vector2f> smoothPos;
        smoothPos.reserve(snake.size());
        for (size_t i = 0; i < snake.size(); i++) {
            sf::Vector2f a = cellCenter((i < prevSnake.size()) ? prevSnake[i] : snake[i]);
            sf::Vector2f b = cellCenter(snake[i]);
            smoothPos.push_back(lerp(a, b, alpha));
        }

        window.clear();
        for (int y = 0; y < gridH; y++) {
            for (int x = 0; x < gridW; x++) {
                tile.setPosition((float)x * cellSize, (float)y * cellSize);
                tile.setFillColor(((x + y) % 2 == 0) ? grassA : grassB);
                window.draw(tile);
            }
        }

        float t = tClock.getElapsedTime().asSeconds();
        float pulse = 1.f + 0.04f * std::sin(t * 3.2f);

        for (auto& a : apples) drawAppleGoogleStyle(a, pulse, t);

        for (size_t i = 0; i + 1 < smoothPos.size(); i++) drawCapsule(smoothPos[i], smoothPos[i + 1], bodyColor);

        if (!smoothPos.empty()) {
            setCircleCenter(headCircle, smoothPos[0]);
            window.draw(headCircle);

            sf::Vector2f hc = smoothPos[0];
            sf::Vector2f forward(0.f, 0.f), side(0.f, 0.f);
            if (moveDir == Right) { forward = { +6.f, 0.f }; side = { 0.f, +6.f }; }
            if (moveDir == Left)  { forward = { -6.f, 0.f }; side = { 0.f, +6.f }; }
            if (moveDir == Down)  { forward = { 0.f, +6.f }; side = { +6.f, 0.f }; }
            if (moveDir == Up)    { forward = { 0.f, -6.f }; side = { +6.f, 0.f }; }

            sf::Vector2f eye1 = hc + forward + side * 0.85f;
            sf::Vector2f eye2 = hc + forward - side * 0.85f;
            sf::Vector2f pupilShift = forward * 0.35f;

            setCircleCenter(eyeWhite, eye1);
            window.draw(eyeWhite);
            setCircleCenter(eyeWhite, eye2);
            window.draw(eyeWhite);

            setCircleCenter(pupil, eye1 + pupilShift);
            window.draw(pupil);
            setCircleCenter(pupil, eye2 + pupilShift);
            window.draw(pupil);
        }

        if (hasFont) {
            tScore.setString("SCORE: " + std::to_string(score));
            tBest.setString("BEST: " + std::to_string(bestScore));
            drawTextShadow(tScore, 12.f, 8.f);
            drawTextShadow(tBest, (float)winW - 190.f, 8.f);
        }

        if (!started && !gameOver) {
            panel.setPosition((float)winW / 2.f, (float)winH / 2.f);
            window.draw(panel);
            if (hasFont) {
                if (hasSave) {
                    tHint.setString("Loaded last game!\nPress SPACE or Arrow Key to start\nP - Pause   R - New Game");
                } else {
                    tHint.setString("Press SPACE or Arrow Key to start\nP - Pause   R - Restart");
                }
                auto b = tHint.getLocalBounds();
                tHint.setOrigin(b.left + b.width / 2.f, b.top + b.height / 2.f);
                tHint.setPosition((float)winW / 2.f, (float)winH / 2.f);
                tHint.setFillColor(sf::Color::White);
                window.draw(tHint);
            }
        }

        if (paused && !gameOver) {
            panel.setPosition((float)winW / 2.f, (float)winH / 2.f);
            window.draw(panel);
            if (hasFont) {
                tPause.setString("PAUSED");
                auto b = tPause.getLocalBounds();
                tPause.setOrigin(b.left + b.width / 2.f, b.top + b.height / 2.f);
                tPause.setPosition((float)winW / 2.f, (float)winH / 2.f - 10.f);
                tPause.setFillColor(sf::Color::White);
                window.draw(tPause);

                tHint.setString("Press P to continue\nR - Restart");
                auto h = tHint.getLocalBounds();
                tHint.setOrigin(h.left + h.width / 2.f, h.top + h.height / 2.f);
                tHint.setPosition((float)winW / 2.f, (float)winH / 2.f + 35.f);
                tHint.setFillColor(sf::Color::White);
                window.draw(tHint);
            }
        }

        if (gameOver) {
            sf::RectangleShape overlay(sf::Vector2f((float)winW, (float)winH));
            overlay.setFillColor(sf::Color(0, 0, 0, 110));
            window.draw(overlay);

            panel.setPosition((float)winW / 2.f, (float)winH / 2.f);
            window.draw(panel);

            if (hasFont) {
                tOver.setString("GAME OVER");
                auto b = tOver.getLocalBounds();
                tOver.setOrigin(b.left + b.width / 2.f, b.top + b.height / 2.f);
                tOver.setPosition((float)winW / 2.f, (float)winH / 2.f - 15.f);
                tOver.setFillColor(sf::Color::White);
                window.draw(tOver);

                tHint.setString("R - Restart\nESC - Quit");
                auto h = tHint.getLocalBounds();
                tHint.setOrigin(h.left + h.width / 2.f, h.top + h.height / 2.f);
                tHint.setPosition((float)winW / 2.f, (float)winH / 2.f + 40.f);
                tHint.setFillColor(sf::Color::White);
                window.draw(tHint);
            }
        }

        window.display();
    }

    // final safety saves
    saveGame(SAVE_PATH, snake, apples, score, stepTime, moveDir, queuedDir);
    saveScores(SCORES_PATH, scores);
    return 0;
}
