// LogicCircuitSimulator.java
// Simple Swing-based logic circuit simulator (visual-free, textual connector + simulation)
// - Add gates and wires via a small UI
// - Evaluate the circuit truth table for N input bits
//
// Compile & run: javac LogicCircuitSimulator.java && java LogicCircuitSimulator

import javax.swing.*;
import java.awt.*;
import java.awt.event.*;
import java.util.*;
import java.util.List;

/**
 * LogicCircuitSimulator
 *
 * A compact Swing UI that allows:
 * - Adding named inputs and gates (AND/OR/NOT/XOR)
 * - Connecting gate inputs using textual wiring (simple syntax)
 * - Producing a truth table and evaluating outputs
 *
 * Wiring format (each line): target = TYPE(arg1,arg2,...)
 * Examples:
 * IN1 = INPUT
 * IN2 = INPUT
 * G1 = AND(IN1,IN2)
 * OUT = OR(G1,IN1)
 */
public class LogicCircuitSimulator extends JFrame {
    private final JTextArea txtWiring = new JTextArea(12, 40);
    private final JTextArea txtOutput = new JTextArea(12, 40);
    private final JButton btnSimulate = new JButton("Simulate (truth table)");

    public LogicCircuitSimulator() {
        super("Logic Circuit Simulator");
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setLayout(new BorderLayout());
        JPanel top = new JPanel(new BorderLayout());
        top.setBorder(BorderFactory.createTitledBorder("Wiring (one statement per line)"));
        txtWiring.setText("A = INPUT\nB = INPUT\nG1 = AND(A,B)\nOUT = OR(G1,A)\n");
        top.add(new JScrollPane(txtWiring), BorderLayout.CENTER);
        add(top, BorderLayout.NORTH);

        JPanel mid = new JPanel(new FlowLayout(FlowLayout.LEFT));
        btnSimulate.addActionListener(e -> simulate());
        mid.add(btnSimulate);
        add(mid, BorderLayout.CENTER);

        JPanel bottom = new JPanel(new BorderLayout());
        bottom.setBorder(BorderFactory.createTitledBorder("Output"));
        txtOutput.setEditable(false);
        bottom.add(new JScrollPane(txtOutput), BorderLayout.CENTER);
        add(bottom, BorderLayout.SOUTH);

        pack();
        setLocationRelativeTo(null);
    }

    static abstract class Node {
        String name;
        Node(String name) { this.name = name; }
        abstract boolean eval(Map<String, Node> env, Map<String, Boolean> cache);
    }

    static class InputNode extends Node {
        InputNode(String name) { super(name); }
        boolean value = false;
        @Override
        boolean eval(Map<String, Node> env, Map<String, Boolean> cache) {
            return value;
        }
    }

    static class GateNode extends Node {
        String type;
        List<String> args = new ArrayList<>();
        GateNode(String name, String type) { super(name); this.type = type.toUpperCase(); }
        @Override
        boolean eval(Map<String, Node> env, Map<String, Boolean> cache) {
            if (cache.containsKey(name)) return cache.get(name);
            List<Boolean> vals = new ArrayList<>();
            for (String a : args) {
                Node n = env.get(a);
                if (n == null) throw new RuntimeException("Undefined node: " + a);
                vals.add(n.eval(env, cache));
            }
            boolean res;
            switch (type) {
                case "AND": res = vals.stream().allMatch(Boolean::booleanValue); break;
                case "OR": res = vals.stream().anyMatch(Boolean::booleanValue); break;
                case "NOT": res = !vals.get(0); break;
                case "XOR":
                    int ones = 0;
                    for (boolean v: vals) if (v) ones++;
                    res = (ones % 2 == 1);
                    break;
                case "NAND": res = !vals.stream().allMatch(Boolean::booleanValue); break;
                case "NOR": res = !vals.stream().anyMatch(Boolean::booleanValue); break;
                default: throw new RuntimeException("Unknown gate type: " + type);
            }
            cache.put(name, res);
            return res;
        }
    }

    private void simulate() {
        try {
            Map<String, Node> env = parseWiring(txtWiring.getText());
            List<String> inputs = new ArrayList<>();
            List<String> outputs = new ArrayList<>();

            for (Map.Entry<String, Node> e : env.entrySet()) {
                if (e.getValue() instanceof InputNode) inputs.add(e.getKey());
            }
            // treat nodes not inputs and not used as intermediate? We'll treat last defined non-input as output(s)
            for (String k : env.keySet()) {
                Node n = env.get(k);
                if (!(n instanceof InputNode)) outputs.add(k);
            }
            // Build truth table
            StringBuilder sb = new StringBuilder();
            sb.append("Inputs: ").append(inputs).append("\n");
            sb.append("Outputs: ").append(outputs).append("\n\n");

            int n = inputs.size();
            int rows = 1 << n;
            for (int mask = 0; mask < rows; mask++) {
                // set inputs
                for (int i = 0; i < n; i++) {
                    boolean bit = ((mask >> (n - i - 1)) & 1) == 1;
                    ((InputNode) env.get(inputs.get(i))).value = bit;
                }
                Map<String, Boolean> cache = new HashMap<>();
                sb.append(String.format("%" + n + "s", Integer.toBinaryString(mask)).replace(' ', '0'));
                sb.append(" | ");
                List<String> outvals = new ArrayList<>();
                for (String out : outputs) {
                    boolean v = env.get(out).eval(env, cache);
                    outvals.add(v ? "1" : "0");
                }
                sb.append(String.join("", outvals));
                sb.append("\n");
            }
            txtOutput.setText(sb.toString());
        } catch (Exception ex) {
            txtOutput.setText("ERROR: " + ex.getMessage());
        }
    }

    private Map<String, Node> parseWiring(String text) {
        Map<String, Node> env = new LinkedHashMap<>();
        String[] lines = text.split("\\r?\\n");
        for (String line : lines) {
            line = line.trim();
            if (line.isEmpty()) continue;
            String[] parts = line.split("=", 2);
            if (parts.length != 2) throw new RuntimeException("Invalid line: " + line);
            String target = parts[0].trim();
            String rhs = parts[1].trim();
            // parse RHS like TYPE(arg1,arg2) or INPUT
            if (rhs.equalsIgnoreCase("INPUT")) {
                env.put(target, new InputNode(target));
            } else {
                int p = rhs.indexOf("(");
                int q = rhs.lastIndexOf(")");
                if (p < 0 || q < 0) throw new RuntimeException("Invalid gate syntax: " + rhs);
                String type = rhs.substring(0, p).trim();
                String argsStr = rhs.substring(p+1, q).trim();
                GateNode g = new GateNode(target, type);
                if (!argsStr.isEmpty()) {
                    for (String a : argsStr.split(",")) g.args.add(a.trim());
                }
                env.put(target, g);
            }
        }
        return env;
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            LogicCircuitSimulator sim = new LogicCircuitSimulator();
            sim.setVisible(true);
        });
    }
}
