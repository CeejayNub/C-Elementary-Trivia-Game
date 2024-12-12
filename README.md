using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading;

class ScoreRecord
{
    public int Score { get; set; }
    public TimeSpan Duration { get; set; }
}

abstract class BaseQuestion
{
    public string Text { get; set; }
    public string[] Options { get; set; }
    public int CorrectAnswer { get; set; }

    public abstract void DisplayQuestion();
}

class TriviaQuestion : BaseQuestion
{
    public Subject Subject { get; set; }
    public Difficulty Difficulty { get; set; }
    public int GradeLevel { get; set; }

    public override void DisplayQuestion()
    {
        // Displaying the subject and question number
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine($"\n{Subject}: {Text}");
        Console.ResetColor();

        // Display choices as A, B, C, D
        for (int j = 0; j < Options.Length; j++)
        {
            char optionLetter = (char)('A' + j); // Generates A, B, C, D
            Console.WriteLine($"{optionLetter}. {Options[j]}");
        }
    }
}

enum Subject
{
    Math,
    English,
    Science,
    Filipino,
    AP,
    MAPEH,
    EPP
}

enum Difficulty
{
    Easy,
    Medium,
    Hard
}

class TriviaGame
{
    private static List<BaseQuestion> questionBank = new List<BaseQuestion>();
    private static string scoreFilePath = "scores.txt";
    private static string basePath = @"C:\Rubio";
    private static string teacherFilePath = Path.Combine(basePath, "teacher_accounts.txt");
    private static string studentFilePath = Path.Combine(basePath, "enrolled_students.txt");
    private static bool questionsLoaded = false;

    // Student info
    private static string studentGradeLevel; // Store enrolled student's grade level
    private static string studentName;
    private static string role;
    private static Dictionary<string, string> teacherCredentials = new Dictionary<string, string>
{
    { "Justine", "Teacher123" }, // Add as many accounts as needed
    { "teacher2", "password2" }
};


    static void Main(string[] args)
    {
        Console.Title = "Elementary Trivia Game"; // Set Console Title
        WelcomeMessage();

        // Collect role (Student or Teacher)
        role = GetRoleFromUser();

        // Load questions from the subject-specific files
        if (!questionsLoaded)
        {
            LoadQuestionsFromFile();
            questionsLoaded = true;
        }

        if (role == "teacher")
        {
            ManageQuestions(); // Allow teacher to manage questions
        }
        if (role == "none")
        {
            Console.WriteLine("Returning to the main menu...");
            Thread.Sleep(2000);
            return; // Prevent further progression
        }
        else
        {

            int gradeLevel = GetGradeLevelFromUser();

            // Check if the selected grade level matches the enrolled student's grade
            if (!studentGradeLevel.Equals($"Grade {gradeLevel}", StringComparison.OrdinalIgnoreCase))
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("You cannot take the quiz for this grade level. You are only allowed to take the quiz for your grade.");
                Console.ResetColor();
                return; // Exit the game if grade levels do not match
            }

            Difficulty difficulty = GetDifficultyFromUser();

            Dictionary<Subject, int> subjectScores = new Dictionary<Subject, int>();
            Dictionary<Subject, TimeSpan> subjectDurations = new Dictionary<Subject, TimeSpan>();

            foreach (Subject subject in Enum.GetValues(typeof(Subject)))
            {
                List<BaseQuestion> questions = LoadQuestions(subject, difficulty, gradeLevel);

                Console.Clear();
                Console.ForegroundColor = ConsoleColor.Yellow;
                Console.WriteLine($"\n Starting Quiz for {subject} ");
                Console.WriteLine(new string('=', 30));
                Console.ResetColor();
                Thread.Sleep(2000); 

                DateTime quizStartTime = DateTime.Now;

                int score = PlayGame(questions);

                DateTime quizEndTime = DateTime.Now;
                TimeSpan duration = quizEndTime - quizStartTime;

                subjectScores[subject] = score;
                subjectDurations[subject] = duration;

                SaveScoreToFile(score, questions.Count, subject, duration, gradeLevel, difficulty);

                if (subject != Enum.GetValues(typeof(Subject)).Cast<Subject>().Last())
                {
                    Console.ForegroundColor = ConsoleColor.Cyan;
                    Console.WriteLine("\n Great job! Get ready for the next subject...");
                    Console.ResetColor();
                    Thread.Sleep(5000); 
                    Console.Clear();
                }
            }

            DisplayHighestAndLowestScores(subjectScores, subjectDurations);
        }

        static string GetRoleFromUser()
        {
            string[] roles = { "Student", "Teacher" };
            int selectedIndex = 0;

            while (true)
            {
                Console.Clear();
                Console.ForegroundColor = ConsoleColor.Magenta;
                Console.WriteLine("===== SELECT YOUR ROLE =====");
                Console.ResetColor();

                for (int i = 0; i < roles.Length; i++)
                {
                    if (i == selectedIndex)
                    {
                        Console.ForegroundColor = ConsoleColor.Yellow;
                        Console.Write(">> ");
                    }
                    else
                    {
                        Console.Write("   ");
                    }
                    Console.WriteLine(roles[i]);
                    Console.ResetColor();
                }

                ConsoleKey key = Console.ReadKey(true).Key;

                if (key == ConsoleKey.UpArrow)
                {
                    selectedIndex = (selectedIndex == 0) ? roles.Length - 1 : selectedIndex - 1;
                }
                else if (key == ConsoleKey.DownArrow)
                {
                    selectedIndex = (selectedIndex == roles.Length - 1) ? 0 : selectedIndex + 1;
                }
                else if (key == ConsoleKey.Enter)
                {
                    if (roles[selectedIndex] == "Student")
                    {
                        if (StudentLogin())
                        {
                            return "student"; 
                        }
                        else
                        {
                            Console.ForegroundColor = ConsoleColor.Red;
                            Console.WriteLine("Access denied. Only enrolled students can play the game.");
                            Console.ResetColor();
                            return "none"; 
                        }
                    }
                    else if (roles[selectedIndex] == "Teacher")
                    {
                        if (TeacherLogin())
                        {
                            return "teacher";
                        }
                        else
                        {
                            Console.ForegroundColor = ConsoleColor.Red;
                            Console.WriteLine("Failed to log in as a teacher. Returning to the main menu...");
                            Console.ResetColor();
                            Thread.Sleep(2000);
                            return "none";
                        }
                    }
                }
            }
        }

        static bool TeacherLogin()
        {
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("===== TEACHER LOGIN =====");
            Console.ResetColor();

            Console.Write("Enter username: ");
            string username = Console.ReadLine().Trim();

            Console.Write("Enter password: ");
            string password = MaskPasswordInput();

            if (File.Exists(teacherFilePath))
            {
                foreach (var line in File.ReadLines(teacherFilePath))
                {
                    var parts = line.Split(',');
                    if (parts[0] == username && parts[1] == password)
                    {
                        Console.ForegroundColor = ConsoleColor.Green;
                        Console.WriteLine("Login successful!");
                        Console.ResetColor();
                        return true;
                    }
                }
            }

            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine("Invalid username or password.");
            Console.ResetColor();
            return false;
        }

        static string MaskPasswordInput()
        {
            string password = "";
            ConsoleKey key;

            do
            {
                var keyInfo = Console.ReadKey(intercept: true);
                key = keyInfo.Key;

                if (key == ConsoleKey.Backspace && password.Length > 0)
                {
                    password = password[..^1];
                    Console.Write("\b \b");
                }
                else if (!char.IsControl(keyInfo.KeyChar))
                {
                    password += keyInfo.KeyChar;
                    Console.Write("*");
                }
            } while (key != ConsoleKey.Enter);

            Console.WriteLine();
            return password;
        }

        static bool StudentLogin()
        {
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("===== STUDENT LOGIN =====");
            Console.ResetColor();

            // Ask for student's full name
            Console.Write("Enter your full name: ");
            string studentName = Console.ReadLine().Trim();

            // Ask for the student's grade level
            Console.Write("Enter your grade level (e.g., Grade 1, Grade 2): ");
            string gradeLevel = Console.ReadLine().Trim();

            bool isEnrolled = false;

            if (File.Exists(studentFilePath))
            {
                foreach (var line in File.ReadLines(studentFilePath))
                {
                    var parts = line.Split(',');

                    if (parts[0].Trim().Equals(studentName, StringComparison.OrdinalIgnoreCase) && parts[1].Trim().Equals(gradeLevel, StringComparison.OrdinalIgnoreCase))
                    {
                        Console.ForegroundColor = ConsoleColor.Green;
                        Console.WriteLine("Student verified. You can play the game!");
                        Console.ResetColor();
                        isEnrolled = true;
                        studentGradeLevel = gradeLevel;
                        break;
                    }
                }
            }

            if (!isEnrolled)
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("Student not enrolled or grade level mismatch. Please contact your teacher.");
                Console.ResetColor();
            }

            return isEnrolled;
        }

        static void EnrollStudent()
        {
            Console.Write("Enter the student's full name: ");
            string studentName = Console.ReadLine().Trim();

            Console.Write("Enter the student's grade level (e.g., Grade 1, Grade 2): ");
            string gradeLevel = Console.ReadLine().Trim();

            if (!string.IsNullOrEmpty(studentName) && !string.IsNullOrEmpty(gradeLevel))
            {
                string enrollmentInfo = $"{studentName},{gradeLevel}";
                File.AppendAllText(studentFilePath, enrollmentInfo + Environment.NewLine);
                Console.ForegroundColor = ConsoleColor.Green;
                Console.WriteLine("Student enrolled successfully!");
                Console.ResetColor();

                string[] options = { "Log in with the newly enrolled student", "Go back to the previous menu" };
                int selectedIndex = 0;

                while (true)
                {
                    Console.Clear();
                    Console.ForegroundColor = ConsoleColor.Cyan;
                    Console.WriteLine("Would you like to:");
                    Console.ResetColor();

                    for (int i = 0; i < options.Length; i++)
                    {
                        if (i == selectedIndex)
                        {
                            Console.ForegroundColor = ConsoleColor.Yellow;
                            Console.Write(">> ");
                        }
                        else
                        {
                            Console.Write("   ");
                        }
                        Console.WriteLine(options[i]);
                        Console.ResetColor();
                    }

                    ConsoleKey key = Console.ReadKey(true).Key;

                    if (key == ConsoleKey.UpArrow)
                    {
                        selectedIndex = (selectedIndex == 0) ? options.Length - 1 : selectedIndex - 1;
                    }
                    else if (key == ConsoleKey.DownArrow)
                    {
                        selectedIndex = (selectedIndex == options.Length - 1) ? 0 : selectedIndex + 1;
                    }
                    else if (key == ConsoleKey.Enter)
                    {
                        if (selectedIndex == 0)
                        {
                            studentName = studentName.Trim(); 
                            studentGradeLevel = gradeLevel.Trim();

                            role = "student";

                            Console.ForegroundColor = ConsoleColor.Green;
                            Console.WriteLine("Logging in with the newly enrolled student...");
                            Console.ResetColor();
                            Thread.Sleep(2000);

                            int gradeLevelSelection = GetGradeLevelFromUser(); 
                            Difficulty difficulty = GetDifficultyFromUser();
                            PlayAllSubjects(gradeLevelSelection, difficulty); 
                            return;
                        }
                        else if (selectedIndex == 1)
                        {
                            return; 
                        }
                    }
                }
            }
            else
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("Student name and grade level cannot be empty.");
                Console.ResetColor();
            }
        }

        static void ViewEnrolledStudents()
        {
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("===== ENROLLED STUDENTS =====");
            Console.ResetColor();

            if (File.Exists(studentFilePath))
            {
                var students = File.ReadLines(studentFilePath).ToList();
                if (students.Count == 0)
                {
                    Console.WriteLine("No students are currently enrolled.");
                }
                else
                {
                    foreach (var student in students)
                    {
                        Console.WriteLine(student);
                    }
                }
            }
            else
            {
                Console.WriteLine("No students enrolled yet. The file does not exist.");
            }

            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine("\nPress any key to return to the menu...");
            Console.ResetColor();
            Console.ReadKey();
        }

        static void RemoveStudent()
        {
            while (true)
            {
                Console.Clear();
                Console.ForegroundColor = ConsoleColor.Cyan;
                Console.WriteLine("\nREMOVE A STUDENT");
                Console.WriteLine("=================");
                Console.ResetColor();

                // Check if there are enrolled students
                if (!File.Exists(studentFilePath) || new FileInfo(studentFilePath).Length == 0)
                {
                    Console.ForegroundColor = ConsoleColor.Yellow;
                    Console.WriteLine("No students are currently enrolled.");
                    Console.ResetColor();
                    Console.WriteLine("\nPress any key to return to the teacher menu...");
                    Console.ReadKey();
                    return;
                }

                // Load students from the file
                List<string> enrolledStudents = File.ReadAllLines(studentFilePath).ToList();
                enrolledStudents.Add("Back to Teacher Menu"); // Add back option

                int currentSelection = 0; // Track the current selection

                while (true)
                {
                    Console.Clear();
                    Console.ForegroundColor = ConsoleColor.Cyan;
                    Console.WriteLine("\nREMOVE A STUDENT");
                    Console.WriteLine("=================");
                    Console.ResetColor();

                    Console.WriteLine("Use the UP and DOWN arrows to navigate, and press ENTER to select.\n");
                    for (int i = 0; i < enrolledStudents.Count; i++)
                    {
                        if (i == currentSelection)
                        {
                            Console.ForegroundColor = ConsoleColor.Yellow;
                            Console.WriteLine($"> {enrolledStudents[i]}");
                        }
                        else
                        {
                            Console.ResetColor();
                            Console.WriteLine($"  {enrolledStudents[i]}");
                        }
                    }
                    Console.ResetColor();

                    // Handle key input
                    ConsoleKeyInfo keyInfo = Console.ReadKey(intercept: true);
                    if (keyInfo.Key == ConsoleKey.UpArrow)
                    {
                        currentSelection = (currentSelection == 0) ? enrolledStudents.Count - 1 : currentSelection - 1;
                    }
                    else if (keyInfo.Key == ConsoleKey.DownArrow)
                    {
                        currentSelection = (currentSelection == enrolledStudents.Count - 1) ? 0 : currentSelection + 1;
                    }
                    else if (keyInfo.Key == ConsoleKey.Enter)
                    {
                        if (currentSelection == enrolledStudents.Count - 1) // Back option selected
                        {
                            return;
                        }
                        else
                        {
                            string removedStudent = enrolledStudents[currentSelection];
                            enrolledStudents.RemoveAt(currentSelection);

                            // Save updated list back to the file
                            File.WriteAllLines(studentFilePath, enrolledStudents.Take(enrolledStudents.Count - 1)); // Exclude "Back to Teacher Menu"

                            Console.ForegroundColor = ConsoleColor.Green;
                            Console.WriteLine($"\nStudent {removedStudent} removed successfully.");
                            Console.ResetColor();

                            Console.WriteLine("\nPress any key to return to the teacher menu...");
                            Console.ReadKey();
                            return;
                        }
                    }
                }
            }
        }



        static void ManageQuestions()
        {
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("\nWelcome Teacher! You can now manage the questions.");
            Console.ResetColor();

            string[] menuOptions = new string[]
            {
                 "View all questions",
                 "Add a new question",
                 "Edit an existing question",
                 "Enroll a student",
                 "View enrolled students",
                 "Remove a student",
                 "Exit"
            };

            int currentSelection = 0; 

            while (true)
            {
                Console.Clear();
                Console.ForegroundColor = ConsoleColor.Magenta;
                Console.WriteLine("What would you like to do?");
                Console.ResetColor();

                for (int i = 0; i < menuOptions.Length; i++)
                {
                    if (i == currentSelection)
                        Console.ForegroundColor = ConsoleColor.Cyan; 
                    else
                        Console.ForegroundColor = ConsoleColor.White;

                    Console.WriteLine(menuOptions[i]);
                }
                Console.ResetColor();

                ConsoleKeyInfo keyInfo = Console.ReadKey(intercept: true);

                if (keyInfo.Key == ConsoleKey.UpArrow)
                {
                    currentSelection = (currentSelection == 0) ? menuOptions.Length - 1 : currentSelection - 1;
                }
                else if (keyInfo.Key == ConsoleKey.DownArrow)
                {
                    currentSelection = (currentSelection == menuOptions.Length - 1) ? 0 : currentSelection + 1;
                }
                else if (keyInfo.Key == ConsoleKey.Enter)
                {
                    switch (currentSelection)
                    {
                        case 0:
                            ViewQuestions();
                            break;
                        case 1:
                            AddQuestion();
                            break;
                        case 2:
                            EditQuestion();
                            break;
                        case 3:
                            EnrollStudent();
                            break;
                        case 4:
                            ViewEnrolledStudents();
                            break;
                        case 5:
                            RemoveStudent(); 
                            break;
                        case 6:
                            Console.ForegroundColor = ConsoleColor.Green;
                            Console.WriteLine("Exiting...");
                            Console.ResetColor();
                            Environment.Exit(0);
                            break;
                        default:
                            break;
                    }
                }
            }
        }

        static void ViewQuestions()
        {
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("\nDisplaying all questions:");
            Console.ResetColor();

            if (questionBank.Count == 0)
            {
                Console.WriteLine("No questions available.");
            }
            else
            {
                foreach (var question in questionBank)
                {
                    question.DisplayQuestion();
                    Console.WriteLine($"Correct Answer: {question.Options[question.CorrectAnswer]}");
                    Console.WriteLine("====================================");
                }
            }

            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine("\nPress any key to return to the menu...");
            Console.ResetColor();
            Console.ReadKey();
        }

        static void AddQuestion()
        {
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("\nAdding a new question:");
            Console.ResetColor();

            TriviaQuestion newQuestion = new TriviaQuestion();

            Console.Write("Enter the question text: ");
            newQuestion.Text = Console.ReadLine().Trim();

            Console.Write("Enter option A: ");
            string optionA = Console.ReadLine().Trim();

            Console.Write("Enter option B: ");
            string optionB = Console.ReadLine().Trim();

            Console.Write("Enter option C: ");
            string optionC = Console.ReadLine().Trim();

            Console.Write("Enter option D: ");
            string optionD = Console.ReadLine().Trim();

            newQuestion.Options = new string[] { optionA, optionB, optionC, optionD };

            // Get correct answer
            while (true)
            {
                Console.Write("Enter the number of the correct answer (1-4): ");
                if (int.TryParse(Console.ReadLine().Trim(), out int correctAnswer) && correctAnswer >= 1 && correctAnswer <= 4)
                {
                    newQuestion.CorrectAnswer = correctAnswer - 1; 
                    break;
                }
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("Invalid input. Please enter a number between 1 and 4.");
                Console.ResetColor();
            }

            // Get subject, difficulty, and grade level
            newQuestion.Difficulty = GetDifficultyFromUser();
            newQuestion.GradeLevel = GetGradeLevelFromUser();

            // Add the new question to the question bank
            questionBank.Add(newQuestion);


            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine("New question added successfully!");
            Console.ResetColor();
        }

        static void EditQuestion()
        {
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("\nEditing an existing question:");
            Console.ResetColor();

            // Display questions
            for (int i = 0; i < questionBank.Count; i++)
            {
                Console.WriteLine($"{i + 1}. {questionBank[i].Text}");
            }

            Console.Write("Enter the number of the question to edit: ");
            if (int.TryParse(Console.ReadLine().Trim(), out int questionNumber) && questionNumber > 0 && questionNumber <= questionBank.Count)
            {
                TriviaQuestion questionToEdit = (TriviaQuestion)questionBank[questionNumber - 1];

                // Ask for new question details (optional)
                Console.Write("Enter new question text (leave blank to keep current): ");
                string newQuestionText = Console.ReadLine().Trim();
                if (!string.IsNullOrEmpty(newQuestionText)) questionToEdit.Text = newQuestionText;

                Console.Write("Enter new option A (leave blank to keep current): ");
                string newOptionA = Console.ReadLine().Trim();
                if (!string.IsNullOrEmpty(newOptionA)) questionToEdit.Options[0] = newOptionA;

                Console.Write("Enter new option B (leave blank to keep current): ");
                string newOptionB = Console.ReadLine().Trim();
                if (!string.IsNullOrEmpty(newOptionB)) questionToEdit.Options[1] = newOptionB;

                Console.Write("Enter new option C (leave blank to keep current): ");
                string newOptionC = Console.ReadLine().Trim();
                if (!string.IsNullOrEmpty(newOptionC)) questionToEdit.Options[2] = newOptionC;

                Console.Write("Enter new option D (leave blank to keep current): ");
                string newOptionD = Console.ReadLine().Trim();
                if (!string.IsNullOrEmpty(newOptionD)) questionToEdit.Options[3] = newOptionD;

                // Update the correct answer
                Console.Write("Enter the new correct answer number (1-4): ");
                if (int.TryParse(Console.ReadLine().Trim(), out int newCorrectAnswer) && newCorrectAnswer >= 1 && newCorrectAnswer <= 4)
                {
                    questionToEdit.CorrectAnswer = newCorrectAnswer - 1; // Store as 0-based index
                }

                Console.ForegroundColor = ConsoleColor.Green;
                Console.WriteLine("Question edited successfully!");
                Console.ResetColor();
            }
            else
            {
                Console.ForegroundColor = ConsoleColor.Red;
                Console.WriteLine("Invalid question number. Please try again.");
                Console.ResetColor();
            }
        }
    }



    static void WelcomeMessage()
        {
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine(@"
  ████████╗██████╗ ██╗██╗   ██╗██╗ █████╗      
  ╚══██╔══╝██╔══██╗██║██║   ██║██║██╔══██╗     
     ██║   ██████╔╝██║██║   ██║██║███████║     
     ██║   ██╔═██╝ ██║╚██╗ ██╔╝██║██╔══██║     
     ██║   ██║ ██  ██║ ╚████╔╝ ██║██║  ██║     
     ╚═╝   ╚═╝ ╚═╝   ╚═╝  ╚═══╝  ╚═╝╚═╝  ╚═╝     
  ===========================================
       WELCOME TO ELEMENTARY TRIVIA GAME          
      Sharpen your knowledge and have fun!
  ===========================================
        ");
            Console.ResetColor();
            Thread.Sleep(2000);
        }
    static int GetGradeLevelFromUser()
    {
        string[] gradeLevels = new string[] { "Grade 1", "Grade 2", "Grade 3", "Grade 4", "Grade 5", "Grade 6" };
        int selectedGradeLevelIndex = 0;

        while (true)
        {
            Console.Clear();
            Console.ForegroundColor = ConsoleColor.Magenta;
            Console.WriteLine("Select your grade level using the arrow keys:");
            Console.ResetColor();

            for (int i = 0; i < gradeLevels.Length; i++)
            {
                if (i == selectedGradeLevelIndex)
                {
                    Console.ForegroundColor = ConsoleColor.Cyan;
                    Console.WriteLine($"> {gradeLevels[i]}");
                    Console.ResetColor();
                }
                else
                {
                    Console.WriteLine($"  {gradeLevels[i]}");
                }
            }

            ConsoleKeyInfo key = Console.ReadKey(intercept: true);

            if (key.Key == ConsoleKey.UpArrow)
            {
                selectedGradeLevelIndex = (selectedGradeLevelIndex == 0) ? gradeLevels.Length - 1 : selectedGradeLevelIndex - 1;
            }
            else if (key.Key == ConsoleKey.DownArrow)
            {
                selectedGradeLevelIndex = (selectedGradeLevelIndex == gradeLevels.Length - 1) ? 0 : selectedGradeLevelIndex + 1;
            }
            else if (key.Key == ConsoleKey.Enter)
            {
                // Return the selected grade level (index + 1 to match Grade 1, Grade 2, etc.)
                return selectedGradeLevelIndex + 1;
            }
        }
    }

    static Difficulty GetDifficultyFromUser()
{
    string[] difficulties = new string[] { "Easy", "Medium", "Hard" };
    int selectedDifficultyIndex = 0;

    while (true)
    {
        Console.Clear();
        Console.ForegroundColor = ConsoleColor.Magenta;
        Console.WriteLine("Select difficulty level using the arrow keys:");
        Console.ResetColor();

        for (int i = 0; i < difficulties.Length; i++)
        {
            if (i == selectedDifficultyIndex)
            {
                Console.ForegroundColor = ConsoleColor.Cyan;
                Console.WriteLine($"> {difficulties[i]}");
                Console.ResetColor();
            }
            else
            {
                Console.WriteLine($"  {difficulties[i]}");
            }
        }

        ConsoleKeyInfo key = Console.ReadKey(intercept: true);

        if (key.Key == ConsoleKey.UpArrow)
        {
            selectedDifficultyIndex = (selectedDifficultyIndex == 0) ? difficulties.Length - 1 : selectedDifficultyIndex - 1;
        }
        else if (key.Key == ConsoleKey.DownArrow)
        {
            selectedDifficultyIndex = (selectedDifficultyIndex == difficulties.Length - 1) ? 0 : selectedDifficultyIndex + 1;
        }
        else if (key.Key == ConsoleKey.Enter)
        {
            // Return the selected difficulty as an enum value
            return (Difficulty)selectedDifficultyIndex;
        }
    }
}

        static void PlayAllSubjects(int gradeLevel, Difficulty difficulty)
{
    Dictionary<Subject, int> subjectScores = new Dictionary<Subject, int>();
    Dictionary<Subject, TimeSpan> subjectDurations = new Dictionary<Subject, TimeSpan>();

    foreach (Subject subject in Enum.GetValues(typeof(Subject)))
    {
        List<BaseQuestion> questions = LoadQuestions(subject, difficulty, gradeLevel);

        Console.Clear();
        Console.ForegroundColor = ConsoleColor.Yellow;
        Console.WriteLine($"\nStarting Quiz for {subject}");
        Console.WriteLine(new string('=', 30));
        Console.ResetColor();
        Thread.Sleep(2000);

        DateTime quizStartTime = DateTime.Now;
        int score = PlayGame(questions);
        DateTime quizEndTime = DateTime.Now;
        TimeSpan duration = quizEndTime - quizStartTime;

        subjectScores[subject] = score;
        subjectDurations[subject] = duration;

        SaveScoreToFile(score, questions.Count, subject, duration, gradeLevel, difficulty);

        if (subject != Enum.GetValues(typeof(Subject)).Cast<Subject>().Last())
        {
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("\nGreat job! Get ready for the next subject...");
            Console.ResetColor();
            Thread.Sleep(3000);
        }
    }

    DisplayHighestAndLowestScores(subjectScores, subjectDurations);
}

        static int PlayGame(List<BaseQuestion> questions)
        {
            Random random = new Random();
            int score = 0;
            int totalQuestions = questions.Count;

            questions = questions.OrderBy(q => random.Next()).ToList();

            for (int i = 0; i < totalQuestions; i++)
            {
                BaseQuestion question = questions[i];
                question.DisplayQuestion(); // Display the question using the overridden method

                // Start timer for the question
                DateTime questionStartTime = DateTime.Now;

                Console.Write("Your answer (A, B, C, D): ");
                string userAnswer = Console.ReadLine().Trim().ToUpper();

                // Calculate answer duration
                TimeSpan questionDuration = DateTime.Now - questionStartTime;

                // Check if the input is a valid letter A-D
                if (userAnswer.Length == 1 && userAnswer[0] >= 'A' && userAnswer[0] <= 'D')
                {
                    int userAnswerIndex = userAnswer[0] - 'A'; // Convert letter to index (A=0, B=1, C=2, D=3)
                    if (userAnswerIndex == question.CorrectAnswer)
                    {
                        Console.ForegroundColor = ConsoleColor.Green;
                        Console.WriteLine(" Correct! Well done!");
                        score++;
                    }
                    else
                    {
                        Console.ForegroundColor = ConsoleColor.Red;
                        Console.WriteLine($"X Wrong! The correct answer is: {question.Options[question.CorrectAnswer]}");
                    }
                }
                else
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine("X Invalid choice. Please select A, B, C, or D.");
                }

                Console.ResetColor();
            }

            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine($"\n===== Game Over! Your final score: {score}/{totalQuestions} =====");
            Console.ResetColor();

            return score;
        }

        static List<BaseQuestion> LoadQuestions(Subject subject, Difficulty difficulty, int gradeLevel)
        {
            return questionBank
                .Where(q => q is TriviaQuestion trivia && trivia.Subject == subject && trivia.Difficulty == difficulty && trivia.GradeLevel == gradeLevel)
                .ToList();
        }

        static void LoadQuestionsFromFile()
        {
            string[] subjects = Enum.GetNames(typeof(Subject));
            foreach (var subject in subjects)
            {
                string filePath = Path.Combine(basePath, $"{subject.ToLower()}_questions.txt");

                // Check if the file exists before attempting to read it
                if (File.Exists(filePath))
                {
                    try
                    {
                        string[] lines = File.ReadAllLines(filePath);
                        foreach (var line in lines)
                        {
                            string[] parts = line.Split('|');
                            if (parts.Length == 8) // Ensure each line contains 8 parts (Question + 4 options + Correct answer + GradeLevel + Difficulty)
                            {
                                string questionText = parts[0].Trim();
                                string[] options = parts[1..5].Select(opt => opt.Trim()).ToArray();
                                int correctAnswerIndex = int.Parse(parts[5].Trim()) - 1; // Correct answer is 1-based, make it 0-based

                                int gradeLevel = int.Parse(parts[6].Trim());
                                Difficulty difficulty = (Difficulty)Enum.Parse(typeof(Difficulty), parts[7].Trim(), true);

                                TriviaQuestion question = new TriviaQuestion
                                {
                                    Text = questionText,
                                    Options = options,
                                    CorrectAnswer = correctAnswerIndex,
                                    Subject = (Subject)Enum.Parse(typeof(Subject), subject, true),
                                    Difficulty = difficulty,
                                    GradeLevel = gradeLevel
                                };
                                questionBank.Add(question);
                            }
                            else
                            {
                                // Skip malformed lines
                                Console.ForegroundColor = ConsoleColor.Yellow;
                                Console.WriteLine($"Skipping malformed line: {line}");
                                Console.ResetColor();
                            }
                        }
                    }
                    catch (IOException ex)
                    {
                        Console.ForegroundColor = ConsoleColor.Red;
                        Console.WriteLine($"Error reading from file: {filePath}. Error: {ex.Message}");
                        Console.ResetColor();
                    }
                    catch (Exception ex)
                    {
                        Console.ForegroundColor = ConsoleColor.Red;
                        Console.WriteLine($"Unexpected error while loading questions from {filePath}: {ex.Message}");
                        Console.ResetColor();
                    }
                }
                else
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine($"File not found: {filePath}");
                    Console.ResetColor();
                }
            }
        }

    static void SaveScoreToFile(int score, int totalQuestions, Subject subject, TimeSpan duration, int gradeLevel, Difficulty difficulty)
    {
        // Create file path for subject-specific scores
        string filePath = Path.Combine(basePath, $"{subject}_scores.txt");

        // Get the current date
        string currentDate = DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss");

        // Format the data to include student name, grade, subject, and score
        string data = $"Student Name: {studentName}, Grade Level: {gradeLevel}, " +
                      $"Subject: {subject}, Difficulty: {difficulty}, " +
                      $"Score: {score}/{totalQuestions}, Duration: {duration.TotalSeconds} seconds, " +
                      $"Date Taken: {currentDate}{Environment.NewLine}";

        // Append the data to the subject-specific file
        File.AppendAllText(filePath, data);
        Console.ForegroundColor = ConsoleColor.Green;
        Console.WriteLine($"Score saved successfully for {subject}!");
        Console.ResetColor();
    }

    static void DisplayHighestAndLowestScores(Dictionary<Subject, int> subjectScores, Dictionary<Subject, TimeSpan> subjectDurations)
        {
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("\n╔══════════════════════════════════╗");
            Console.WriteLine("║           GAME RESULTS          ║");
            Console.WriteLine("╚══════════════════════════════════╝");
            Console.ResetColor();

            // Find the highest and lowest scores
            var highestScoreSubject = subjectScores.OrderByDescending(s => s.Value).First();
            var lowestScoreSubject = subjectScores.OrderBy(s => s.Value).First();

            // Display highest score
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine("\n HIGHEST SCORE ");
            Console.WriteLine("╔══════════════════════════════╗");
            Console.WriteLine($"║ Subject: {highestScoreSubject.Key,-16} ║");
            Console.WriteLine($"║ Score:   {highestScoreSubject.Value,-16} ║");
            Console.WriteLine("╚══════════════════════════════╝");
            Console.ResetColor();

            // Display lowest score
            Console.ForegroundColor = ConsoleColor.Red;
            Console.WriteLine("\n LOWEST SCORE ");
            Console.WriteLine("╔══════════════════════════════╗");
            Console.WriteLine($"║ Subject: {lowestScoreSubject.Key,-16} ║");
            Console.WriteLine($"║ Score:   {lowestScoreSubject.Value,-16} ║");
            Console.WriteLine("╚══════════════════════════════╝");
            Console.ResetColor();

            // Find subjects that need improvement (score <= 1, can be adjusted based on criteria)
            var subjectsNeedingImprovement = subjectScores.Where(s => s.Value <= 1).Select(s => s.Key).ToList();
            string needImprovement = subjectsNeedingImprovement.Any()
                ? string.Join(", ", subjectsNeedingImprovement)
                : "None";

            Console.ForegroundColor = ConsoleColor.Yellow;
            Console.WriteLine("\n NEED IMPROVEMENT ");
            Console.WriteLine($"Subjects: {needImprovement}");
            Console.ResetColor();

            // Display detailed scores for each subject
            Console.ForegroundColor = ConsoleColor.Cyan;
            Console.WriteLine("\n DETAILED SCORES ");
            Console.WriteLine("═══════════════════════════════════");
            foreach (var subject in subjectScores)
            {
                Console.WriteLine($" {subject.Key}: {subject.Value} points ( {subjectDurations[subject.Key].TotalSeconds:F1}s)");
            }
            Console.WriteLine("═══════════════════════════════════");
            Console.ResetColor();
        }
    }
