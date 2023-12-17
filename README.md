import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
 
//основная программа
public class Main extends JFrame {
 
    //справка
    public class Refer implements ActionListener {
        JFrame frame;
 
        @Override
        public void actionPerformed(ActionEvent e) {
            if (frame != null) {
                frame.dispose();
            }
            frame = new JFrame();
            frame.setVisible(true);
            frame.setSize(new Dimension(500, 280));
            frame.setTitle("Справка");
            frame.setLayout(null);
            JLabel about = new JLabel(
                    "<html>" +
                            "<p>'Начальная станция' - выбирается станция от которой начинается формирование кратчайшего пути;</p><br>" +
                            "<p>'Конечная станция' - выбирается станция до которой формируется кратчайший путь;</p><br>" +
                            "<p>'Общее время в пути' - длина кратчайшего пути в минутах;</p><br>" +
                            "<p>Кнопка 'Поиск' - отображает кратчайший путь красной линией на рисунке и добавляет его время в минутах на метку 'Общее время в пути';</p><br>" +
                            "<p>Кнопка 'Сброс' - сбрасывает начальную и конечную станции, а также убирает ранее найденный кратчайший путь;</p><br>" +
                    "</html>");
            about.setBounds(20, 20, 450, 220);
            frame.add(about);
            frame.setDefaultCloseOperation(JFrame.DISPOSE_ON_CLOSE);
            frame.setResizable(false);
        }
    }
 
    //конструктор
    public Main() {
 
        //настройка формы
        setResizable(false);
        setTitle("Схема метро");
        int w = 656;
        int h = 600;
        setPreferredSize(new java.awt.Dimension(w, h));
        GroupLayout layout = new javax.swing.GroupLayout(getContentPane());
        getContentPane().setLayout(layout);
        pack();
        setDefaultCloseOperation(javax.swing.WindowConstants.EXIT_ON_CLOSE);
 
        //о программе
        JMenuBar menuBar = new JMenuBar();
        JMenu menu = new JMenu("О программе");
 
        //справка
        JMenuItem refer = new JMenuItem("Справка");
        refer.addActionListener(new Refer());
        menu.add(refer);
 
        //размещаем меню
        menuBar.add(menu);
        setJMenuBar(menuBar);
 
        //добавляем панель
        JPanel panel = new Panel();
        panel.setBounds(0, 0, w, h);
        panel.setLayout(null);
        add(panel);
    }
 
    //главная функция
    public static void main(String[] args) {
        new Main().setVisible(true);
    }
}

import javax.swing.*;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.io.BufferedReader;
import java.io.FileReader;
import java.util.ArrayList;
import java.util.Collections;
import java.util.Hashtable;
 
import static javax.swing.JOptionPane.showMessageDialog;
 
//интерфейс
public class Panel extends JPanel implements ActionListener {
    Hashtable<String, Integer> dict; //словарь станций
    ArrayList<Station> stations; //массив для отрисовки
    int[][] graph; //матрица смежности
 
    //получаем данные о станциях из файла (матрица смежности)
    void readGraph(String path) {
        try {
            BufferedReader reader = new BufferedReader(new FileReader(path));
            String line = reader.readLine(); //размер графа
            int size = Integer.parseInt(line);
            graph = new int[size][size];
            dict.clear();
            int k = 0;
 
            //по каждой вершине графа
            for (int i = 0; i < size; i++) {
 
                //формат строки:
                // название тек. ст. -> смежные ст. метро - время прибытия от тек. ст.
                line = reader.readLine();
                String[] lines = line.split(" ");
                String name1 = lines[0];
                if (!dict.containsKey(name1)) {
                    dict.put(name1, k++); //присваиваем уник. номер
                }
 
                //цикл по каждой смежной вершине
                for (int j = 1; j < lines.length; j++) {
                    String[] vert = lines[j].split("-");
                    if (vert.length == 2) {
 
                        //получаем смежную вершину и её вес
                        String name2 = vert[0];
                        int min = Integer.parseInt(vert[1]);
                        if (!dict.containsKey(name2)) {
                            dict.put(name2, k++); //присваиваем уник. номер
                        }
 
                        //граф неориентированный
                        graph[dict.get(name1)][dict.get(name2)] = min;
                        graph[dict.get(name2)][dict.get(name1)] = min;
                    } else {
                        throw new Exception("Вершина графа должна иметь вес!");
                    }
                }
            }
            reader.close();
        } catch (Exception e) {
            showMessageDialog(null, e.getMessage(),
                    "Ошибка!", JOptionPane.INFORMATION_MESSAGE);
        }
    }
 
    //получаем список станций и их местоположение для отрисовки
    void readStations(String path) {
        try {
            BufferedReader reader = new BufferedReader(new FileReader(path));
            String line = reader.readLine(); //кол-во вершин
            int size = Integer.parseInt(line);
            stations.clear();
            for (int i = 0; i < size; i++) {
                line = reader.readLine();
 
                //формат строки: название позиция по X позиция по Y цвет вершины
                String[] lines = line.split(" ");
                if (lines.length == 4) {
                    Point p = new Point(Integer.parseInt(lines[1]), Integer.parseInt(lines[2]));
                    String c = lines[3];
                    Color color = Color.green;
                    if (c.equals("m")) {
                        color = Color.magenta;
                    } else if (c.equals("r")) {
                        color = Color.red;
                    } else if (c.equals("b")) {
                        color = Color.blue;
                    } else if (c.equals("o")) {
                        color = Color.orange;
                    }
                    Station station = new Station(lines[0], p, color, dict.get(lines[0]));
                    stations.add(station);
                } else {
                    throw new Exception("Неверно указан формат станции метро!");
                }
            }
            reader.close();
        } catch (Exception e) {
            showMessageDialog(null, e.getMessage(),
                    "Ошибка!", JOptionPane.INFORMATION_MESSAGE);
        }
    }
 
    //поиск пути из start в end
    //алгоритм Дейкстры
    public ArrayList<Integer> findPath(int start, int end) {
        int inf = 10000;
        int n = stations.size();
        int[] d = new int[n];
        boolean[] used = new boolean[n];
        int[] p = new int[n];
        for (int i = 0; i < n; i++) {
            d[i] = inf;
            used[i] = false;
        }
        d[start] = 0;
        p[start] = -1;
        for (int i = 0; i < n; i++) {
 
            //находим следующую непосещенную вершину
            int v = -1;
            for (int j = 0; j < n; j++) {
                if (!used[j] && (v == -1 || d[j] < d[v])) {
                    v = j;
                }
            }
 
            //вершина недостижима
            if (d[v] == inf) {
                break;
            }
            used[v] = true;
 
            //проводим релаксации
            for (int j = 0; j < n; j++) {
                if (graph[v][j] != 0) {
                    if (d[v] + graph[v][j] < d[j]) {
                        d[j] = d[v] + graph[v][j];
                        p[j] = v;
                    }
                }
            }
        }
 
        //восстанавливаем путь
        ArrayList<Integer> path = new ArrayList<>();
        for (int v = end; v != -1; v = p[v]) {
            path.add(v);
        }
        Collections.reverse(path);
        return path;
    }
 
    //конструктор
    public Panel() {
 
        //загрузка данных из файлов
        dict = new Hashtable<>();
        stations = new ArrayList<>();
        readGraph("graph.txt");
        readStations("stations.txt");
        if (dict.size() != stations.size()) {
            showMessageDialog(null, "Количество вершин для " +
                            "отрисовки и для расчета " +
                            "не совпадают!",
                    "Внимание!", JOptionPane.INFORMATION_MESSAGE);
        }
 
        //панель для отрисовки станций метро
        int w = 600;
        int h = 400;
        int paddX = 40; //отступ слева
        Visual panel = new Visual(w, h, stations, graph);
        panel.setBounds(20, 20, w, h);
        panel.setBackground(Color.WHITE);
        add(panel);
 
        //установка начальной станции
        JLabel lb1 = new JLabel("Начальная станция:");
        JComboBox combo1 = new JComboBox(dict.keySet().toArray());
        lb1.setBounds(paddX + 20, h + 40, 200, 20);
        combo1.setBounds(paddX + 20, h + 70, 150, 20);
        add(lb1);
        add(combo1);
 
        //установка конечной станции
        JLabel lb2 = new JLabel("Конечная станция:");
        JComboBox combo2 = new JComboBox(dict.keySet().toArray());
        lb2.setBounds(paddX + 200, h + 40, 200, 20);
        combo2.setBounds(paddX + 200, h + 70, 150, 20);
        add(lb2);
        add(combo2);
 
        //примерное время
        JLabel lb3 = new JLabel("Общее время в пути:");
        lb3.setBounds(paddX + 20, h + 100, 200, 20);
        add(lb3);
 
        //найти кратчайший путь
        JButton find = new JButton("Поиск");
        find.addActionListener((e) -> {
            String start = combo1.getSelectedItem().toString();
            String end = combo2.getSelectedItem().toString();
            ArrayList<Integer> path = findPath(dict.get(start), dict.get(end));
            int d = 0;
            for(int i = 0; i < path.size() - 1; i++){
                d += graph[path.get(i)][path.get(i + 1)];
            }
            lb3.setText("Общее время в пути: " + d + " минут");
            panel.setPath(path);
        });
        find.setBounds(paddX + 420, h + 50, 100, 20);
        add(find);
 
        //сброс поиска
        JButton reset = new JButton("Сброс");
        reset.addActionListener((e) -> {
            combo1.setSelectedIndex(0);
            combo2.setSelectedIndex(0);
            lb3.setText("Общее время в пути:");
            panel.reset();
        });
        reset.setBounds(paddX + 420, h + 80, 100, 20);
        add(reset);
    }
 
    @Override
    public void actionPerformed(ActionEvent e) {
        repaint();
    }
}

import javax.swing.*;
import java.awt.*;
import java.util.ArrayList;
import java.util.Hashtable;
 
//панель для отрисовки
public class Visual extends JPanel {
    private int w, h; //размеры панели
 
    private Hashtable<Integer, Station> dict; //словарь для быстрого доступа к данным
    private int[][] graph; //матрица смежности
    private ArrayList<Integer> path; //кратчайший маршрут
 
    //конструктор
    public Visual(int w, int h, ArrayList<Station> stations, int[][] graph) {
        this.w = w;
        this.h = h;
        this.graph = graph;
        dict = new Hashtable<>();
        for (int i = 0; i < stations.size(); i++) {
            dict.put(stations.get(i).index, stations.get(i));
        }
        setPath(new ArrayList<>());
    }
 
    //установка кр. маршрута
    public void setPath(ArrayList<Integer> path) {
        this.path = path;
        repaint();
    }
 
    //отрисовка компонентов
    @Override
    public void paintComponent(Graphics g) {
        super.paintComponent(g);
        drawGraph(g);
    }
 
    //сброс
    public void reset() {
        setPath(new ArrayList<>());
        repaint();
    }
 
    //отрисовка графа
    void drawGraph(Graphics g) {
        int paddX = -25; //смещение по x
        int d = 15; //диаметр круга
 
        //пути для каждой вершины по матрице смежности
        g.setColor(Color.BLACK);
        for (int i = 0; i < graph.length; i++) {
            for (int j = 0; j < graph[i].length; j++) {
                if (graph[i][j] != 0) {
                    Station start = dict.get(i);
                    Station end = dict.get(j);
 
                    //ребро
                    g.drawLine(start.pos.x + d / 2 + paddX, start.pos.y + d / 2,
                            end.pos.x + d / 2 + paddX, end.pos.y + d / 2);
 
                    //вес ребра
                    int mx = (start.pos.x + end.pos.x) / 2 + d / 2 + paddX;
                    int my = (start.pos.y + end.pos.y) / 2 + d / 2;
                    g.drawString(graph[i][j] + "", mx, my - 2);
                }
            }
        }
 
        //кратчайший путь
        if (path.size() > 1) {
            g.setColor(Color.red);
            for (int i = 0; i < path.size() - 1; i++) {
                Station start = dict.get(path.get(i));
                Station end = dict.get(path.get(i + 1));
                g.drawLine(start.pos.x + d / 2 + paddX, start.pos.y + d / 2,
                        end.pos.x + d / 2 + paddX, end.pos.y + d / 2);
            }
        }
 
        //названия и подписи
        for (int index : dict.keySet()) {
            Station s = dict.get(index);
            g.setColor(s.color);
            g.fillOval(s.pos.x + paddX, s.pos.y, d, d);
            g.drawString(s.name, s.pos.x + paddX, s.pos.y - 6);
        }
    }
}

import java.awt.*;
 
//одна станция метро
public class Station {
    public String name; //название
    public Point pos; //позиция при отрисовке
    public Color color; //цвет
    public int index; //для идентификации
 
    //конструктор
    public Station(String name, Point pos, Color color, int index) {
        this.name = name;
        this.pos = pos;
        this.color = color;
        this.index = index;
    }
}
