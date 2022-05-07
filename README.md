[8:27 PM, 5/7/2022] eriny: .
[9:57 PM, 5/7/2022] Kero Rafat: using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.IO;
using System.Threading.Tasks;
using System.Data;
namespace Simple_Shell
{
    public static class Commands
    {
        public static void clear(List<Token> tokens)
        {
            if (tokens.Count > 1)
            {
                Console.WriteLine("Error: " + tokens[0].value + " command syntax is \n cls \n function: Clear the screen.");
            }
            else
            {
                Console.Clear();
            }
        }
        public static void quit(List<Token> tokens)
        {
            if (tokens.Count > 1)
            {
                Console.WriteLine("Error: " + tokens[0].value + " command syntax is \n quit \n function: Quit the shell.");
            }
            else
            {
                FAT.writeFAT();
                Virtual_Disk.Disk.Close();
                Environment.Exit(0);
            }
        }
        public static void createDirectory(List<Token> tokens)
        {
            if (tokens.Count <= 1 || tokens.Count > 2)
            {
                Console.WriteLine("Error: " + tokens[0].value + "command syntax is \n md [directory]\n[directory] can be a new directory name or fullpath of a new directory\nCreates a directory.");
            }
            else
            {
                if (tokens[1].key == TokenType.DirName)
                {
                    if (Program.current.searchDirectory(tokens[1].value) == -1)
                    {
                        if (FAT.getAvilableCluster() != -1)
                        {
                            Directory_Entry d = new Directory_Entry(tokens[1].value, 0x10, 0);
                            Program.current.DirOrFiles.Add(d);
                            Program.current.writeDirectory();
                            if (Program.current.parent != null)
                            {
                                Program.current.parent.updateContent(Program.current.GetDirectory_Entry());
                                Program.current.parent.writeDirectory();
                            }
                            FAT.writeFAT();
                        }
                        else
                            Console.WriteLine($"Error : sorry the disk is full!");
                    }
                    else
                        Console.WriteLine($"Error : this directory \" {tokens[1].value} \" is already exists!");
                }
                else if (tokens[1].key == TokenType.FullPathToDirectory)
                {
                    Directory dir = moveTodir(tokens[1], false);
                    if (dir == null)
                        Console.WriteLine($"Error : this path \" {tokens[1].value} \" is not exists!");
                    else
                    {
                        if (FAT.getAvilableCluster() != -1)
                        {
                            string[] ss = tokens[1].value.Split('\\');
                            Directory_Entry d = new Directory_Entry(ss[ss.Length - 1], 0x10, 0);
                            dir.DirOrFiles.Add(d);
                            dir.writeDirectory();
                            dir.parent.updateContent(dir.GetDirectory_Entry());
                            dir.parent.writeDirectory();
                            FAT.writeFAT();
                        }
                        else
                            Console.WriteLine($"Error : sorry the disk is full!");
                    }
                }
                else
                {
                    Console.WriteLine("Error: " + tokens[0].value + "command syntax is \n md [directory]\n[directory] can be a new directory name or fullpath of a new directory\nCreates a directory.");
                }
            }
        }
        public static File_Entry createFile(Token token)
        {
            if (token.key == TokenType.FileName)
            {
                if (Program.current.searchDirectory(token.value) == -1)
                {
                    if (FAT.getAvilableCluster() != -1)
                    {
                        Directory_Entry d = new Directory_Entry(token.value, 0x0, 0);
                        Program.current.DirOrFiles.Add(d);
                        Program.current.writeDirectory();
                        if (Program.current.parent != null)
                        {
                            Program.current.parent.updateContent(Program.current.GetDirectory_Entry());
                            Program.current.parent.writeDirectory();
                        }
                        FAT.writeFAT();
                        File_Entry f = new File_Entry(token.value, 0x0, 0, Program.current);
                        return f;
                    }
                    else
                        Console.WriteLine($"Error : sorry the disk is full!");
                }
                else
                    Console.WriteLine($"Error : this file name \" {token.value} \" is already exists!");
            }
            else if (token.key == TokenType.FullPathToFile)
            {
                Token t = new Token();
                t.value = token.value.Substring(0, token.value.LastIndexOf('\\'));
                t.key = token.key;
                Directory dir = moveTodir(t, false);
                if (dir == null)
                    Console.WriteLine($"Error : this path \" {token.value} \" is not exists!");
                else
                {
                    if (FAT.getAvilableCluster() != -1)
                    {
                        string[] ss = token.value.Split('\\');
                        Directory_Entry d = new Directory_Entry(ss[ss.Length - 1], 0x10, 0);
                        dir.DirOrFiles.Add(d);
                        dir.writeDirectory();
                        dir.parent.updateContent(dir.GetDirectory_Entry());
                        dir.parent.writeDirectory();
                        FAT.writeFAT();
                        File_Entry f = new File_Entry(ss[ss.Length - 1], 0x0, 0, dir);
                        return f;
                    }
                    else
                        Console.WriteLine($"Error : sorry the disk is full!");
                }
            }
            return null;
        }
        private static Directory moveTodir(Token token, bool changedirFlag)
        {
            Directory d = null;
            string path;
            if (token.key == TokenType.DirName)
            {
                if (token.value != "..")
                {
                    int i = Program.current.searchDirectory(token.value);
                    if (i == -1)
                        return null;
                    else
                    {
                        string n = new string(Program.current.DirOrFiles[i].dir_name);
                        byte at = Program.current.DirOrFiles[i].dir_attr;
                        int fc = Program.current.DirOrFiles[i].dir_firstCluster;
                        d = new Directory(n, at, fc, Program.current);
                        d.readDirectory();
                        path = Program.currentPath;
                        path += "\\" + n.Trim(new char[] { '\0', ' ' });
                        if (changedirFlag)
                            Program.currentPath = path;
                    }
                }
                else
                {
                    if (Program.current.parent != null)
                    {
                        d = Program.current.parent;
                        d.readDirectory();
                        path = Program.currentPath;
                        path = path.Substring(0, path.LastIndexOf('\\'));
                        if (changedirFlag)
                            Program.currentPath = path;
                    }
                    else
                    {
                        d = Program.current;
                        d.readDirectory();
                    }
                }
            }
            else if (token.key == TokenType.FullPathToDirectory)
            {
                string[] s = token.value.Split('\\');
                List<string> ss = new List<string>();
                for (int i = 0; i < s.Length; i++)
                    if (s[i] != "")
                        ss.Add(s[i]);
                Directory rootd = new Directory("K:", 0x10, 5, null);
                rootd.readDirectory();
                if (ss[0].ToLower().Equals("k:"))
                {
                    path = "k:";
                    int ll = (changedirFlag) ? ss.Count : ss.Count - 1;
                    for (int i = 1; i < ll; i++)
                    {
                        int j = rootd.searchDirectory(ss[i]);
                        if (j != -1)
                        {
                            Directory pa = rootd;
                            string n = new string(rootd.DirOrFiles[j].dir_name);
                            byte at = rootd.DirOrFiles[j].dir_attr;
                            int fc = rootd.DirOrFiles[j].dir_firstCluster;
                            rootd = new Directory(n, at, fc, pa);
                            rootd.readDirectory();
                            path += "\\" + n.Trim(new char[] { '\0', ' ' });
                        }
                        else
                        {
                            return null;
                        }
                    }
                    d = rootd;
                    if (changedirFlag)
                        Program.currentPath = path;
                }
                else if (ss[0] == "..")
                {
                    d = Program.current;
                    for (int i = 0; i < ss.Count; i++)
                    {
                        if (d.parent != null)
                        {
                            d = d.parent;
                            d.readDirectory();
                            path = Program.currentPath;
                            path = path.Substring(0, path.LastIndexOf('\\'));
                            if (changedirFlag)
                                Program.currentPath = path;
                        }
                        else
                        {
                            d = Program.current;
                            d.readDirectory();
                            break;
                        }
                    }
                }
                else
                    return null;
            }
            return d;
        }
        public static void changeDirectory(List<Token> tokens)
        {
            if (tokens.Count == 1)
                Console.WriteLine(Program.currentPath);
            else if (tokens.Count == 2)
            {
                if (tokens[1].key == TokenType.DirName || tokens[1].key == TokenType.FullPathToDirectory)
                {
                    Directory dir = moveTodir(tokens[1], true);
                    if (dir != null)
                    {
                        dir.readDirectory();
                        Program.current = dir;
                    }
                    else
                    {
                        Console.WriteLine($"Error : this path \" {tokens[1].value} \" is not exists!");
                    }
                }
                else
                    Console.WriteLine("Error: " + tokens[0].value + "command syntax is \n cd \n or \n cd [directory]\n[directory] can be a new directory name or fullpath of a directory");
            }
            else
                Console.WriteLine("Error: " + tokens[0].value + "command syntax is \n cd \n or \n cd [directory]\n[directory] can be a new directory name or fullpath of a directory");
        }
        public static void help(List<Token> tokens)
        {
            if (tokens.Count > 2)
            {
                Console.WriteLine("Error: " + tokens[0].value + " command syntax is \n help \n or \n help [command] \n function:Provides Help information for commands.");
            }
            else if (tokens.Count == 2)
            {
                if (tokens[1].key == TokenType.Command)
                {
                    switch (tokens[1].value)
                    {
                        case "cd":
                            Console.WriteLine("Change the current default directory to the directory given in the argument.");
                            Console.WriteLine("If the argument is not present, report the current directory.");
                            Console.WriteLine("If the directory does not exist an appropriate error should be reported.");
                            Console.WriteLine(tokens[1].value + " command syntax is \n cd \n or \n cd [directory]");
                            Console.WriteLine("[directory] can be directory name or fullpath of a directory");
                            break;
                        case "cls":
                            Console.WriteLine("Clear the screen.");
                            Console.WriteLine(tokens[1].value + " command syntax is \n cls");
                            break;
                        case "dir":
                            Console.WriteLine("List the contents of directory given in the argument.");
                            Console.WriteLine("If the argument is not present, list the content of the current directory.");
                            Console.WriteLine("If the directory does not exist an appropriate error should be reported.");
                            Console.WriteLine(tokens[1].value + " command syntax is \n dir \n or \n dir [directory]");
                            Console.WriteLine("[directory] can be directory name or fullpath of a directory");
                            break;
                        case "quit":
                            Console.WriteLine("Quit the shell.");
                            Console.WriteLine(tokens[1].value + " command syntax is \n quit");
                            break;
                        case "copy":
                            Console.WriteLine("Copies one or more files to another location.");
                            Console.WriteLine(tokens[1].value + " command syntax is \n copy [source]+ [destination]");
                            Console.WriteLine("+ after [source] represent that you can pass more than file Name (or fullpath of file) or more than directory Name (or fullpath of directory)");
                            Console.WriteLine("[source] can be file Name (or fullpath of file) or directory Name (or fullpath of directory)");
                            Console.WriteLine("[destination] can be directory name or fullpath of a directory");
                            break;
                        case "del":
                            Console.WriteLine("Deletes one or more files.");
                            Console.WriteLine("NOTE: it confirms the user choice to delete the file before deleting");
                            Console.WriteLine(tokens[1].value + " command syntax is \n del [file]+");
                            Console.WriteLine("+ after [file] represent that you can pass more than file Name (or fullpath of file)");
                            Console.WriteLine("[file] can be file Name (or fullpath of file)");
                            break;
                        case "help":
                            Console.WriteLine("Provides Help information for commands.");
                            Console.WriteLine(tokens[1].value + " command syntax is \n help \n or \n For more information on a specific command, type help [command]");
                            Console.WriteLine("command - displays help information on that command.");
                            break;
                        case "md":
                            Console.WriteLine("Creates a directory.");
                            Console.WriteLine(tokens[1].value + " command syntax is \n md [directory]");
                            Console.WriteLine("[directory] can be a new directory name or fullpath of a new directory");
                            break;
                        case "rd":
                            Console.WriteLine("Removes a directory.");
                            Console.WriteLine("NOTE: it confirms the user choice to delete the directory before deleting");
                            Console.WriteLine(tokens[1].value + " command syntax is \n rd [directory]");
                            Console.WriteLine("[directory] can be a directory name or fullpath of a directory");
                            break;
                        case "rename":
                            Console.WriteLine("Renames a file.");
                            Console.WriteLine(tokens[1].value + " command syntax is \n rd [fileName] [new fileName]");
                            Console.WriteLine("[fileName] can be a file name or fullpath of a filename ");
                            Console.WriteLine("[new fileName] can be a new file name not fullpath ");
                            break;
                        case "type":
                            Console.WriteLine("Displays the contents of a text file.");
                            Console.WriteLine(tokens[1].value + " command syntax is \n type [file]");
                            Console.WriteLine("NOTE: it displays the filename before its content for every file");
                            Console.WriteLine("[file] can be file Name (or fullpath of file) of text file");
                            break;
                        case "import":
                            Console.WriteLine("– import text file(s) from your computer ");
                            Console.WriteLine(tokens[1].value + " command syntax is \n import [destination] [file]+");
                            Console.WriteLine("+ after [file] represent that you can pass more than file Name (or fullpath of file) of text file");
                            Console.WriteLine("[file] can be file Name (or fullpath of file) of text file");
                            Console.WriteLine("[destination] can be directory name or fullpath of a directory in your implemented file system");
                            break;
                        case "export":
                            Console.WriteLine("– export text file(s) to your computer ");
                            Console.WriteLine(tokens[1].value + " command syntax is \n export [destination] [file]+");
                            Console.WriteLine("+ after [file] represent that you can pass more than file Name (or fullpath of file) of text file");
                            Console.WriteLine("[file] can be file Name (or fullpath of file) of text file in your implemented file system");
                            Console.WriteLine("[destination] can be directory name or fullpath of a directory in your computer");
                            break;
                    }
                }
                else
                {
                    Console.WriteLine("Error: " + tokens[1].value + " => This command is not supported by the help utility.");
                }
            }
            else if (tokens.Count == 1)
            {
                Console.WriteLine("cd       - Change the current default directory to .");
                Console.WriteLine("           If the argument is not present, report the current directory.");
                Console.WriteLine("           If the directory does not exist an appropriate error should be reported.");
                Console.WriteLine("cls      - Clear the screen.");
                Console.WriteLine("dir      - List the contents of directory .");
                Console.WriteLine("quit     - Quit the shell.");
                Console.WriteLine("copy     - Copies one or more files to another location");
                Console.WriteLine("del      - Deletes one or more files.");
                Console.WriteLine("help     - Provides Help information for commands.");
                Console.WriteLine("md       - Creates a directory.");
                Console.WriteLine("rd       - Removes a directory.");
                Console.WriteLine("rename   - Renames a file.");
                Console.WriteLine("type     - Displays the contents of a text file.");
                Console.WriteLine("import   – import text file(s) from your computer");
                Console.WriteLine("export   – export text file(s) to your computer");
            }
        }

        public static void listDirectory(List<Token> tokens)
        {
            if (tokens.Count == 1)
            {
                int filecount = 0;
                int directorycount = 0;
                int filesSizesSum = 0;
                Console.WriteLine(" Directory of " + Program.currentPath);
                Console.WriteLine();
                int start = 1;
                if (Program.current.parent != null)
                {
                    start = 2;
                    Console.WriteLine("{0}{1:11}", "\t<DIR>    ", ".");
                    directorycount++;
                    Console.WriteLine("{0}{1:11}", "\t<DIR>    ", "..");
                    directorycount++;
                }
                for (int i = start; i < Program.current.DirOrFiles.Count; i++)
                {
                    if (Program.current.DirOrFiles[i].dir_attr == 0x0)
                    {
                        Console.WriteLine("\t{0:9}{1:11}", Program.current.DirOrFiles[i].dir_filesize, new string(Program.current.DirOrFiles[i].dir_name));
                        filecount++;
                        filesSizesSum += Program.current.DirOrFiles[i].dir_filesize;
                    }
                    else if (Program.current.DirOrFiles[i].dir_attr == 0x10)
                    {
                        Console.WriteLine("{0}{1:11}", "\t<DIR>    ", new string(Program.current.DirOrFiles[i].dir_name));
                        directorycount++;
                    }
                }
                Console.WriteLine($"{"              "}{filecount} File(s)    {filesSizesSum} bytes");
                Console.WriteLine($"{"              "}{directorycount} Dir(s)    {Virtual_Disk.getFreeSpace()} bytes free");
            }
            else if (tokens.Count == 2)
            {
                if (tokens[1].key == TokenType.DirName || tokens[1].key == TokenType.FullPathToDirectory)
                {
                    Directory dir = moveTodir(tokens[1], false);
                    if (dir != null)
                    {
                        dir.readDirectory();
                        int filecount = 0;
                        int directorycount = 0;
                        int filesSizesSum = 0;
                        if (tokens[1].key == TokenType.DirName)
                        {
                            Console.WriteLine(" Directory of " + Program.currentPath + "\\" + tokens[1].value);
                        }
                        else
                        {
                            Console.WriteLine(" Directory of " + tokens[1].value);
                        }
                        Console.WriteLine();
                        int start = 1;
                        if (dir.parent != null)
                        {
                            start = 2;
                            Console.WriteLine("{0}{1:11}", "\t<DIR>    ", ".");
                            directorycount++;
                            Console.WriteLine("{0}{1:11}", "\t<DIR>    ", "..");
                            directorycount++;
                        }
                        for (int i = start; i < dir.DirOrFiles.Count; i++)
                        {
                            if (dir.DirOrFiles[i].dir_attr == 0x0)
                            {
                                Console.WriteLine("\t{0:9} {1:11}", dir.DirOrFiles[i].dir_filesize, new string(dir.DirOrFiles[i].dir_name));
                                filecount++;
                                filesSizesSum += dir.DirOrFiles[i].dir_filesize;
                            }
                            else if (dir.DirOrFiles[i].dir_attr == 0x10)
                            {
                                Console.WriteLine("{0}{1:11}", "\t<DIR>    ", new string(dir.DirOrFiles[i].dir_name));
                                directorycount++;
                            }
                        }
                        Console.WriteLine($"{"              "}{filecount} File(s)    {filesSizesSum} bytes");
                        Console.WriteLine($"{"              "}{directorycount} Dir(s)    {Virtual_Disk.getFreeSpace()} bytes free");
                    }
                    else
                    {
                        Console.WriteLine($"Error : this path \" {tokens[1].value} \" is not exists!");
                    }
                }
                else
                    Console.WriteLine("Error: " + tokens[0].value + " command syntax is \n dir \n or \n dir [directory]\n[directory] can be a new directory name or fullpath of a directory");
            }
            else
                Console.WriteLine("Error: " + tokens[0].value + " command syntax is \n dir \n or \n dir [directory]\n[directory] can be a new directory name or fullpath of a directory");
        }
        public static void removeDirectory(List<Token> tokens)
        {
            if (tokens.Count <= 1 || tokens.Count > 2)
            {
                Console.WriteLine("Error: " + tokens[0].value + "command syntax is \n rd [directory]\n[directory] can be a new directory name or fullpath of a new directory\nCreates a directory.");
            }
            else
            {
                if (tokens[1].key == TokenType.DirName || tokens[1].key == TokenType.FullPathToDirectory)
                {
                    Directory dir = moveTodir(tokens[1], false);
                    if (dir != null)
                    {
                        Console.Write($"Are you sure that you want to delete {new string(dir.dir_name).Trim(new char[] { '\0', ' ' })} , please enter Y for yes or N for no:");
                        string s = Console.ReadLine().ToLower();
                        if (s.Equals("y"))
                            dir.deleteDirectory();
                    }
                    else
                        Console.WriteLine($"Error : this directory \" {tokens[1].value} \" is not exists!");
                }
                else
                {
                    Console.WriteLine("Error: " + tokens[0].value + "command syntax is \n rd [directory]\n[directory] can be a new directory name or fullpath of a new directory\nCreates a directory.");
                }
            }
        }
        public static void copy()
        {
            string source = Console.ReadLine();
            string target = Console.ReadLine();
            File.Copy(source, target, true);
            //string directoryPath = Path.GetDirectoryName(destpath);

            // If directory doesn't exist create one
            //if (!Directory.Exists(directoryPath))
            //{
            //    DirectoryInfo di = Directory.CreateDirectory(directoryPath);
            //}

            //File.Copy(sourcepath, destpath);
            //Console.WriteLine("File is Copyed");
        }
        static void Rename()
        {

            string filename = Console.ReadLine();
            // Create a FileInfo  
            System.IO.FileInfo fi = new System.IO.FileInfo(filename);
            if (fi.Exists)
            {
                // Move file with a new name  
                fi.MoveTo(Console.ReadLine());
                Console.WriteLine("File Renamed.");
            }
            else
            {
                File.Create(filename);
                Console.WriteLine("File is Not Exict And Will be Created ");
            }

        }
        static void type()
        {
            string textfile = Console.ReadLine();
            // Read a text file line by line.  
            string[] lines = File.ReadAllLines(textfile);

            foreach (string line in lines)
                Console.WriteLine(line);
        }

        static void del()
        {
            string[] files = System.IO.Directory.GetFiles(Console.ReadLine());
            string myfile = Console.ReadLine();

            // Delete one file
            Console.WriteLine("Do You Want To Delete This File Only ? y/n");
            string sure = Console.ReadLine();
            if (sure == "y")
            {
                File.Delete(myfile);
                Console.WriteLine("Specified file has been deleted");
            }
            else
            {
                Console.WriteLine("Are You Sure To Delete All Files ? y/n");
                if (sure == "y")
                {
                    //Delete All files
                    foreach (string file in files)
                    {

                        File.Delete(file);
                        Console.WriteLine($"{file} is deleted.");
                    }
                }
            }
        }
        static void importCommand()
        {
            DataTable table = new DataTable();
            string[] lines = File.ReadAllLines(@"D:\D:\del\o.txt");
            string[] values;


            for (int i = 0; i < lines.Length; i++)
            {
                values = lines[i].ToString().Split('|');
                string[] row = new string[values.Length];

                for (int j = 0; j < values.Length; j++)
                {
                    row[j] = values[j].Trim();
                }
                table.Rows.Add(row);
            }
        }






    }
}
  
  ***************************************************************************************************************************************************
  
  using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Simple_Shell
{
    public static class Parser
    {
        public static void parse(string input)
        {
            List<Token> tokens = Tokenizer.GetTokens(input);
            if (tokens != null)
            {
                if (tokens[0].key == TokenType.Not_Recognized)
                    Console.WriteLine(tokens[0].value + " is not recognized as an internal or external command,operable program or batch file.");
                else
                {
                    if (tokens[0].key == TokenType.Command)
                    {
                        switch (tokens[0].value)
                        {
                            case "cd":
                                Commands.changeDirectory(tokens);
                                break;
                            case "cls":
                                Commands.clear(tokens);
                                break;
                            case "dir":
                                Commands.listDirectory(tokens);
                                break;
                            case "quit":
                                Commands.quit(tokens);
                                break;
                            case "copy":
                                break;
                            case "del":
                                break;
                            case "help":
                                Commands.help(tokens);
                                break;
                            case "md":
                                Commands.createDirectory(tokens);
                                break;
                            case "rd":
                                Commands.removeDirectory(tokens);
                                break;
                            case "rename":
                                break;
                            case "type":
                                break;
                            case "import":
                                break;
                            case "export":
                                break;
                        }
                    }
                }
            }
        }

    }
}
  
  *******************************************************************************************************************************
  
  
  
  
