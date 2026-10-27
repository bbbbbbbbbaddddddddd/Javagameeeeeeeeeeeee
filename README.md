# Javagameeeeeeeeeeeee
import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Random;

interface GameConstants {
    int WINDOW_WIDTH = 800;
    int WINDOW_HEIGHT = 600;

    int PLAYER_WIDTH = 60;
    int PLAYER_HEIGHT = 30;
    int PLAYER_SPEED = 8;
    int PLAYER_Y = WINDOW_HEIGHT - 80;
    Color PLAYER_COLOR = new Color(0, 200, 255);

    int ENEMY_SIZE = 40;
    int ENEMY_BASE_SPEED = 3;
    int ENEMY_SPAWN_INTERVAL = 25;
    Color ENEMY_COLOR = new Color(255, 80, 80);

    int FPS = 60;
    long FRAME_TIME_MS = 1000 / FPS;

    Color BG_COLOR = new Color(20, 20, 35);
    Color TEXT_COLOR = new Color(230, 230, 230);
    Color ACCENT_COLOR = new Color(255, 200, 0);
}

class GameLogic {
    private int playerX;
    private final List<Rectangle> enemies = new ArrayList<>();
    private final Random random = new Random();
    private int frameCounter = 0;
    private int score = 0;
    private boolean gameOver = false;
    private int enemySpeed = GameConstants.ENEMY_BASE_SPEED;

    public GameLogic() { reset(); }

    public void reset() {
        playerX = (GameConstants.WINDOW_WIDTH - GameConstants.PLAYER_WIDTH) / 2;
        enemies.clear();
        frameCounter = 0;
        score = 0;
        gameOver = false;
        enemySpeed = GameConstants.ENEMY_BASE_SPEED;
    }

    public void movePlayer(int dx) {
        if (gameOver) return;
        playerX += dx;
        clampPlayerX();
    }

    public void setPlayerX(int x) {
        if (gameOver) return;
        playerX = x - GameConstants.PLAYER_WIDTH / 2;
        clampPlayerX();
    }

    private void clampPlayerX() {
        if (playerX < 0) playerX = 0;
        if (playerX > GameConstants.WINDOW_WIDTH - GameConstants.PLAYER_WIDTH)
            playerX = GameConstants.WINDOW_WIDTH - GameConstants.PLAYER_WIDTH;
    }

    public void update() {
        if (gameOver) return;
        frameCounter++;
        if (frameCounter % 300 == 0) enemySpeed++;

        if (frameCounter % GameConstants.ENEMY_SPAWN_INTERVAL == 0) {
            int x = random.nextInt(GameConstants.WINDOW_WIDTH - GameConstants.ENEMY_SIZE);
            enemies.add(new Rectangle(x, -GameConstants.ENEMY_SIZE,
                    GameConstants.ENEMY_SIZE, GameConstants.ENEMY_SIZE));
        }

        Rectangle playerRect = new Rectangle(playerX, GameConstants.PLAYER_Y,
                GameConstants.PLAYER_WIDTH, GameConstants.PLAYER_HEIGHT);

        for (int i = enemies.size() - 1; i >= 0; i--) {
            Rectangle enemy = enemies.get(i);
            enemy.y += enemySpeed;
            if (enemy.y > GameConstants.WINDOW_HEIGHT) {
                enemies.remove(i);
                score += 10;
                continue;
            }
            if (enemy.intersects(playerRect)) {
                gameOver = true;
                return;
            }
        }
    }

    public int getPlayerX() { return playerX; }
    public List<Rectangle> getEnemies() { return enemies; }
    public int getScore() { return score; }
    public boolean isGameOver() { return gameOver; }
}

class GamePanel extends JPanel implements Runnable, KeyListener, MouseMotionListener {
    private final GameLogic logic;
    private Thread gameThread;
    private volatile boolean running = false;

    public GamePanel() {
        logic = new GameLogic();
        setPreferredSize(new Dimension(GameConstants.WINDOW_WIDTH, GameConstants.WINDOW_HEIGHT));
        setBackground(GameConstants.BG_COLOR);
        setFocusable(true);
        addKeyListener(this);
        addMouseMotionListener(this);
    }

    public void startGame() {
        if (gameThread == null || !running) {
            running = true;
            gameThread = new Thread(this);
            gameThread.start();
        }
    }

    @Override
    public void run() {
        while (running) {
            long start = System.currentTimeMillis();
            logic.update();
            repaint();
            long elapsed = System.currentTimeMillis() - start;
            long sleep = GameConstants.FRAME_TIME_MS - elapsed;
            if (sleep > 0) {
                try { Thread.sleep(sleep); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }
        }
    }

    @Override
    protected void paintComponent(Graphics g) {
        super.paintComponent(g);
        Graphics2D g2 = (Graphics2D) g;
        g2.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);

        int px = logic.getPlayerX();
        int py = GameConstants.PLAYER_Y;
        int pw = GameConstants.PLAYER_WIDTH;
        int ph = GameConstants.PLAYER_HEIGHT;
        int[] xPoints = {px, px + pw / 2, px + pw};
        int[] yPoints = {py + ph, py, py + ph};
        g2.setColor(GameConstants.PLAYER_COLOR);
        g2.fillPolygon(xPoints, yPoints, 3);

        g2.setColor(GameConstants.ENEMY_COLOR);
        for (Rectangle r : logic.getEnemies())
            g2.fillRoundRect(r.x, r.y, r.width, r.height, 8, 8);

        g2.setColor(GameConstants.TEXT_COLOR);
        g2.setFont(new Font("Monospaced", Font.BOLD, 24));
        g2.drawString("SCORE: " + logic.getScore(), 20, 40);

        if (logic.isGameOver()) {
            g2.setColor(GameConstants.ACCENT_COLOR);
            g2.setFont(new Font("Monospaced", Font.BOLD, 48));
            String msg = "GAME OVER";
            FontMetrics fm = g2.getFontMetrics();
            int msgX = (GameConstants.WINDOW_WIDTH - fm.stringWidth(msg)) / 2;
            g2.drawString(msg, msgX, GameConstants.WINDOW_HEIGHT / 2);

            g2.setFont(new Font("Monospaced", Font.PLAIN, 20));
            String hint = "Press R to restart";
            fm = g2.getFontMetrics();
            int hintX = (GameConstants.WINDOW_WIDTH - fm.stringWidth(hint)) / 2;
            g2.drawString(hint, hintX, GameConstants.WINDOW_HEIGHT / 2 + 50);
        }
    }

    @Override
    public void keyPressed(KeyEvent e) {
        int key = e.getKeyCode();
        if (key == KeyEvent.VK_LEFT || key == KeyEvent.VK_A) logic.movePlayer(-GameConstants.PLAYER_SPEED);
        else if (key == KeyEvent.VK_RIGHT || key == KeyEvent.VK_D) logic.movePlayer(GameConstants.PLAYER_SPEED);
        else if (key == KeyEvent.VK_R) logic.reset();
    }

    @Override public void keyReleased(KeyEvent e) {}
    @Override public void keyTyped(KeyEvent e) {}
    @Override public void mouseMoved(MouseEvent e) { logic.setPlayerX(e.getX()); }
    @Override public void mouseDragged(MouseEvent e) { logic.setPlayerX(e.getX()); }
}

public class DodgeGame {
    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            JFrame frame = new JFrame("Dodge — Java Swing Game");
            frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
            frame.setResizable(false);

            GamePanel panel = new GamePanel();
            frame.add(panel);
            frame.pack();
            frame.setLocationRelativeTo(null);
            frame.setVisible(true);

            panel.startGame();
        });
    }
}

