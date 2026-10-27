
import java.sql.*;
import java.util.Scanner;

public class StudentEnrollment {

    static final String URL =
            "jdbc:mysql://localhost:3306/college_db";
    static final String USER = "root";
    static final String PASSWORD = "your_password";

    static Connection con;
    static Scanner sc = new Scanner(System.in);

    public static void main(String[] args) {
        try {
            con = DriverManager.getConnection(
                    URL, USER, PASSWORD);

            System.out.println("Connected to database!");

            int choice;

            do {
                System.out.println("\n--- Student Enrollment System ---");
                System.out.println("1. Create Course");
                System.out.println("2. Add Student");
                System.out.println("3. Enroll Student");
                System.out.println("4. Display Students in a Course");
                System.out.println("5. Exit");
                System.out.print("Enter choice: ");

                choice = sc.nextInt();
                sc.nextLine();

                switch (choice) {
                    case 1:
                        createCourse();
                        break;
                    case 2:
                        addStudent();
                        break;
                    case 3:
                        enrollStudent();
                        break;
                    case 4:
                        displayStudents();
                        break;
                    case 5:
                        System.out.println("Exiting...");
                        break;
                    default:
                        System.out.println("Invalid choice!");
                }
            } while (choice != 5);

            con.close();

        } catch (SQLException e) {
            System.out.println("Database error: "
                    + e.getMessage());
        }
    }

    // Create a new course
    static void createCourse() throws SQLException {
        System.out.print("Enter course name: ");
        String name = sc.nextLine();

        String sql =
                "INSERT INTO courses (course_name) VALUES (?)";

        try (PreparedStatement ps =
                     con.prepareStatement(sql)) {
            ps.setString(1, name);
            ps.executeUpdate();
            System.out.println("Course created successfully!");
        }
    }

    // Add a student
    static void addStudent() throws SQLException {
        System.out.print("Enter student ID: ");
        int id = sc.nextInt();
        sc.nextLine();

        System.out.print("Enter student name: ");
        String name = sc.nextLine();

        String sql =
                "INSERT INTO students (student_id, student_name) "
                        + "VALUES (?, ?)";

        try (PreparedStatement ps =
                     con.prepareStatement(sql)) {
            ps.setInt(1, id);
            ps.setString(2, name);
            ps.executeUpdate();
            System.out.println("Student added successfully!");
        }
    }

    // Enroll a student in a course
    static void enrollStudent() throws SQLException {
        System.out.print("Enter student ID: ");
        int studentId = sc.nextInt();

        System.out.print("Enter course ID: ");
        int courseId = sc.nextInt();

        // Check whether the student exists
        if (!exists("students", "student_id", studentId)) {
            System.out.println("Student does not exist!");
            return;
        }

        // Check whether the course exists
        if (!exists("courses", "course_id", courseId)) {
            System.out.println("Course does not exist!");
            return;
        }

        String checkSql =
                "SELECT 1 FROM enrollments "
                        + "WHERE student_id = ? AND course_id = ?";

        try (PreparedStatement check =
                     con.prepareStatement(checkSql)) {
            check.setInt(1, studentId);
            check.setInt(2, courseId);

            try (ResultSet rs = check.executeQuery()) {
                if (rs.next()) {
                    System.out.println(
                            "Duplicate enrollment! Student is "
                                    + "already enrolled in this course.");
                    return;
                }
            }
        }

        String sql =
                "INSERT INTO enrollments (student_id, course_id) "
                        + "VALUES (?, ?)";

        try (PreparedStatement ps =
                     con.prepareStatement(sql)) {
            ps.setInt(1, studentId);
            ps.setInt(2, courseId);
            ps.executeUpdate();
            System.out.println("Enrollment successful!");
        } catch (SQLIntegrityConstraintViolationException e) {
            // The database primary key also protects against
            // duplicate enrollments during concurrent requests.
            System.out.println(
                    "Enrollment already exists or violates "
                            + "a database constraint.");
        }
    }

    // Check whether a student or course exists
    static boolean exists(String table, String column, int id)
            throws SQLException {

        // Table and column names are fixed internal values.
        String sql = "SELECT 1 FROM " + table
                + " WHERE " + column + " = ?";

        try (PreparedStatement ps =
                     con.prepareStatement(sql)) {
            ps.setInt(1, id);

            try (ResultSet rs = ps.executeQuery()) {
                return rs.next();
            }
        }
    }

    // Display all students enrolled in a selected course
    static void displayStudents() throws SQLException {
        System.out.print("Enter course ID: ");
        int courseId = sc.nextInt();

        String sql =
                "SELECT s.student_id, s.student_name, "
                        + "c.course_name "
                        + "FROM students s "
                        + "JOIN enrollments e "
                        + "ON s.student_id = e.student_id "
                        + "JOIN courses c "
                        + "ON e.course_id = c.course_id "
                        + "WHERE c.course_id = ? "
                        + "ORDER BY s.student_id";

        try (PreparedStatement ps =
                     con.prepareStatement(sql)) {
            ps.setInt(1, courseId);

            try (ResultSet rs = ps.executeQuery()) {
                System.out.println("\nStudents in Course:");

                boolean found = false;

                while (rs.next()) {
                    found = true;
                    System.out.println(
                            "Student ID: "
                                    + rs.getInt("student_id")
                                    + ", Name: "
                                    + rs.getString("student_name")
                                    + ", Course: "
                                    + rs.getString("course_name"));
                }

                if (!found) {
                    System.out.println(
                            "No students enrolled in this course.");
                }
            }
        }
    }
}
