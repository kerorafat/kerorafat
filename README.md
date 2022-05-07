using System;
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
    
    
    
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    
    
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
    
    
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    
    
    using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.IO;

namespace Simple_Shell
{
    public static class Virtual_Disk
    {

        public static FileStream Disk;
        public static void CREATEorOPEN_Disk(string path)
        {
            Disk = new FileStream(path, FileMode.OpenOrCreate, FileAccess.ReadWrite);
        }
        public static int getFreeSpace()
        {
            return (1024 * 1024) - (int)Disk.Length;
        }
        public static void initalize(string path)
        {
            try
            {
                if (!File.Exists(path))
                {
                    CREATEorOPEN_Disk(path);
                    byte[] b = new byte[1024];
                    for (int i = 0; i < b.Length; i++)
                        b[i] = 0;
                    writeCluster(b, 0);
                    FAT.createFAT();
                    Directory root = new Directory("K:", 0x10, 5, null);
                  
                    root.writeDirectory();
                    FAT.setClusterPointer(5, -1);
                    Program.current = root;
                    FAT.writeFAT();
                }
                else
                {
                    CREATEorOPEN_Disk(path);
                    FAT.readFAT();
                    Directory root = new Directory("K:", 0x10, 5, null);
                    root.readDirectory();
                    Program.current = root;
                }
            }
            catch (IOException ex)
            {
                Console.WriteLine(ex.Message);
            }
        }
        public static void writeCluster(byte[] cluster, int clusterIndex, int offset = 0, int count = 1024)
        {
            Disk.Seek(clusterIndex * 1024, SeekOrigin.Begin);
            Disk.Write(cluster, offset, count);
            Disk.Flush();
        }
        public static byte[] readCluster(int clusterIndex)
        {
            Disk.Seek(clusterIndex * 1024, SeekOrigin.Begin);
            byte[] bytes = new byte[1024];
            Disk.Read(bytes, 0, 1024);
            return bytes;
        }
    }
}
                                                 
                                                 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
                                                 
                                                 
                                                 
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Simple_Shell
{
    class FAT
    {
        public static int[] Fat = new int[1024];
        public static void createFAT()
        {
            for (int i = 0; i < Fat.Length; i++)
            {
                if (i == 0 || i == 4)
                {
                    Fat[i] = -1;
                }
                else if (i > 0 && i <= 3)
                {
                    Fat[i] = i + 1;
                }
                else
                {
                    Fat[i] = 0;
                }
            }
        }
        public static void writeFAT()
        {
            byte[] FATBYTES = Converter.ToBytes(FAT.Fat);
            List<byte[]> ls = Converter.splitBytes(FATBYTES);
            for (int i = 0; i < ls.Count; i++)
            {
                Virtual_Disk.writeCluster(ls[i], i + 1, 0, ls[i].Length);
            }
        }
        public static void readFAT()
        {
            List<byte> ls = new List<byte>();
            for (int i = 1; i <= 4; i++)
            {
                ls.AddRange(Virtual_Disk.readCluster(i));
            }
            Fat = Converter.ToInt(ls.ToArray());
        }
        public static void printFAT()
        {
            Console.WriteLine("FAT has the following: ");
            for (int i = 0; i < Fat.Length; i++)
                Console.WriteLine("FAT[" + i + "] = " + Fat[i]);
        }
        public static void setFAT(int[] arr)
        {
            if (arr.Length <= 1024)
                Fat = arr;
        }
        public static int getAvilableCluster()
        {
            for (int i = 0; i < Fat.Length; i++)
            {
                if (Fat[i] == 0)
                    return i;
            }
            return -1;//our disk is full
        }
        public static void setClusterPointer(int clusterIndex, int pointer)
        {
            Fat[clusterIndex] = pointer;
        }
        public static int getClusterPointer(int clusterIndex)
        {
            if (clusterIndex >= 0 && clusterIndex < Fat.Length)
                return Fat[clusterIndex];
            else
                return -1;
        }
    }
}
                                                               
                                                               ++++++++++++++++++++++++++++++++++++++++
                                                               
                                                               
       using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Simple_Shell
{
    public static class Tokenizer
    {
        static Token generateToken(string arg, TokenType tokenType)
        {
            Token token;
            token.key = tokenType;
            token.value = arg;
            return token;
        }
        static bool checkArg(string arg)
        {
            if (arg == "cd" || arg == "cls" || arg == "dir" || arg == "quit"
                || arg == "copy" || arg == "del" || arg == "help"
                || arg == "md" || arg == "rd" || arg == "rename"
                || arg == "type" || arg == "import" || arg == "export")
            {
                return true;
            }
            return false;
        }
        static bool isFullPathd(string arg)
        {
            if ((arg.Contains(":") || arg.Contains("\\")) && !arg.Contains('.'))
            {
                return true;
            }
            return false;
        }
        static bool isFullPathf(string arg)
        {
            if ((arg.Contains(":") || arg.Contains("\\")) && arg.Contains('.'))
            {
                return true;
            }
            return false;
        }
        static bool isFileName(string arg)
        {
            if (arg.Contains('.')/*&&!arg.Contains("..")*/)
            {
                return true;
            }
            return false;
        }
        public static List<Token> GetTokens(string input)
        {
            List<Token> Tokens = new List<Token>();

            if (input.Length == 0)
                return null;
            string[] inputs = input.Split(' ');
            List<string> ls = new List<string>();
            for (int i = 0; i < inputs.Length; i++)
                if (inputs[i] != "" && inputs[i] != " ")
                    ls.Add(inputs[i]);
            string[] arguments = ls.ToArray();
            arguments[0] = arguments[0].ToLower();
            switch (arguments[0])
            {
                case "cd":
                    if (arguments.Length == 1)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    }
                    else if (arguments.Length == 2)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        if (isFullPathd(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FullPathToDirectory));
                        else if (isFullPathf(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FullPathToFile));
                        else if (isFileName(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FileName));
                        else
                            Tokens.Add(generateToken(arguments[1], TokenType.DirName));
                    }
                    else
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        for (int i = 1; i < arguments.Length; i++)
                        {
                            Tokens.Add(generateToken(arguments[i], TokenType.Not_Recognized));
                        }
                    }
                    break;
                case "cls":
                    Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    if (arguments.Length > 1)
                    {
                        for (int i = 1; i < arguments.Length; i++)
                        {
                            Tokens.Add(generateToken(arguments[i], TokenType.Not_Recognized));
                        }
                    }
                    break;
                case "dir":
                    if (arguments.Length == 1)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    }
                    else if (arguments.Length == 2)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        if (isFullPathd(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FullPathToDirectory));
                        else if (isFullPathf(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FullPathToFile));
                        else if (isFileName(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FileName));
                        else
                            Tokens.Add(generateToken(arguments[1], TokenType.DirName));
                    }
                    else
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        for (int i = 1; i < arguments.Length; i++)
                        {
                            Tokens.Add(generateToken(arguments[i], TokenType.Not_Recognized));
                        }
                    }
                    break;
                case "quit":
                    if (arguments.Length == 1)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    }
                    else
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        for (int i = 1; i < arguments.Length; i++)
                        {
                            Tokens.Add(generateToken(arguments[i], TokenType.Not_Recognized));
                        }
                    }
                    break;
                case "copy":
                    Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    break;
                case "del":
                    Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    break;
                case "help":
                    if (arguments.Length == 1)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    }
                    else if (arguments.Length == 2)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        arguments[1] = arguments[1].ToLower();
                        if (checkArg(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.Command));
                        else
                            Tokens.Add(generateToken(arguments[1], TokenType.Not_Recognized));
                    }
                    else
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        for (int i = 1; i < arguments.Length; i++)
                        {
                            Tokens.Add(generateToken(arguments[i], TokenType.Not_Recognized));
                        }
                    }
                    break;
                case "md":
                    if (arguments.Length == 1)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    }
                    else if (arguments.Length == 2)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        if (isFullPathd(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FullPathToDirectory));
                        else if (isFullPathf(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FullPathToFile));
                        else if (isFileName(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FileName));
                        else
                            Tokens.Add(generateToken(arguments[1], TokenType.DirName));
                    }
                    else
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        for (int i = 1; i < arguments.Length; i++)
                        {
                            Tokens.Add(generateToken(arguments[i], TokenType.Not_Recognized));
                        }
                    }
                    break;
                case "rd":
                    if (arguments.Length == 1)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    }
                    else if (arguments.Length == 2)
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        if (isFullPathd(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FullPathToDirectory));
                        else if (isFullPathf(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FullPathToFile));
                        else if (isFileName(arguments[1]))
                            Tokens.Add(generateToken(arguments[1], TokenType.FileName));
                        else
                            Tokens.Add(generateToken(arguments[1], TokenType.DirName));
                    }
                    else
                    {
                        Tokens.Add(generateToken(arguments[0], TokenType.Command));
                        for (int i = 1; i < arguments.Length; i++)
                        {
                            Tokens.Add(generateToken(arguments[i], TokenType.Not_Recognized));
                        }
                    }
                    break;
                case "rename":
                    Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    break;
                case "type":
                    Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    break;
                case "import":
                    Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    break;
                case "export":
                    Tokens.Add(generateToken(arguments[0], TokenType.Command));
                    break;
                default:
                    Tokens.Add(generateToken(arguments[0], TokenType.Not_Recognized));
                    break;

            }
            return Tokens;
        }
    }
}
    ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    
    using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.IO;

namespace Simple_Shell
{
    public static class Converter
    {
        public static byte[] ToBytes(int[] array)
        {
            byte[] bytes = null;
            bytes = new byte[array.Length * sizeof(int)];
            System.Buffer.BlockCopy(array, 0, bytes, 0, bytes.Length);
            return bytes;
        }
        public static byte[] StringToBytes(string s)
        {
            byte[] bytes = new byte[s.Length];
            for (int i = 0; i < s.Length; i++)
            {
                bytes[i] = (byte)s[i];
            }
            return bytes;
        }
        public static string BytesToString(byte[] bytes)
        {
            string s = string.Empty;
            for (int i = 0; i < bytes.Length; i++)
            {
                if ((char)bytes[i] != '\0')
                    s += (char)bytes[i];
                else
                    break;
            }
            return s;
        }
        public static byte[] Directory_EntryToBytes(Directory_Entry d)
        {
            byte[] bytes = new byte[32];
            for (int i = 0; i < d.dir_name.Length; i++)
            {
                bytes[i] = (byte)d.dir_name[i];
            }
            bytes[11] = d.dir_attr;
            int j = 12;
            for (int i = 0; i < d.dir_empty.Length; i++)
            {
                bytes[j] = d.dir_empty[i];
                j++;
            }
            byte[] fc = BitConverter.GetBytes(d.dir_firstCluster);
            for (int i = 0; i < fc.Length; i++)
            {
                bytes[j] = fc[i];
                j++;
            }
            byte[] sz = BitConverter.GetBytes(d.dir_filesize);
            for (int i = 0; i < sz.Length; i++)
            {
                bytes[j] = sz[i];
                j++;
            }
            return bytes;
         
        }


        public static Directory_Entry BytesToDirectory_Entry(Byte[] bytes)
        {
            char[] name = new char[11];
            for (int i = 0; i < name.Length; i++)
            {
                name[i] = (char)bytes[i];
            }
            byte attr = bytes[11];
            byte[] empty = new byte[12];
            int j = 12;
            for (int i = 0; i < empty.Length; i++)
            {
                empty[i] = bytes[j];
                j++;
            }
            byte[] fc = new byte[4];
            for (int i = 0; i < fc.Length; i++)
            {
                fc[i] = bytes[j];
                j++;
            }
            int firstcluster = BitConverter.ToInt32(fc, 0);
            byte[] sz = new byte[4];
            for (int i = 0; i < sz.Length; i++)
            {
                sz[i] = bytes[j];
                j++;
            }
            int filesize = BitConverter.ToInt32(sz, 0);
            Directory_Entry d = new Directory_Entry(new string(name), attr, firstcluster);
            d.dir_empty = empty;
            d.dir_filesize = filesize;
            return d;
          
        }
        public static byte[] ToBytes(List<Directory_Entry> array)
        {
            using (MemoryStream stream = new MemoryStream())
            {
                var bformatter = new System.Runtime.Serialization.Formatters.Binary.BinaryFormatter();

                bformatter.Serialize(stream, array);
                byte[] arr = stream.ToArray();
                return arr;
            }
        }
        public static int[] ToInt(byte[] bytes)
        {
            int[] ints = null;
            ints = new int[bytes.Length / sizeof(int)];
        

            System.Buffer.BlockCopy(bytes, 0, ints, 0, bytes.Length);
            return ints;
        }
        public static List<Directory_Entry> ToDirectory_Entry(byte[] bytes)
        {
          
            using (MemoryStream stream = new MemoryStream(bytes))
            {
                stream.Seek(0, SeekOrigin.Begin);
                var bformatter = new System.Runtime.Serialization.Formatters.Binary.BinaryFormatter();
                List<Directory_Entry> ls = ((List<object>)bformatter.Deserialize(stream)).Cast<Directory_Entry>().ToList();
                return ls;
            }
        }
        public static List<byte[]> splitBytes(byte[] bytes)
        {
            List<byte[]> ls = new List<byte[]>();
            int number_of_arrays = bytes.Length / 1024;
            int rem = bytes.Length % 1024;
            for (int i = 0; i < number_of_arrays; i++)
            {
                byte[] b = new byte[1024];
                for (int j = i * 1024, k = 0; k < 1024; j++, k++)
                {
                    b[k] = bytes[j];
                }
                ls.Add(b);
            }
            if (rem > 0)
            {
                byte[] b1 = new byte[1024];
                for (int i = number_of_arrays * 1024, k = 0; k < rem; i++, k++)
                {
                    b1[k] = bytes[i];
                }
                ls.Add(b1);
            }
            return ls;
        }
    }
}
                                                                     
                                                                     +++++++++++++++++++++++++++++++++++++
 
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Simple_Shell
{
    [Serializable]
    public class Directory_Entry
    {
        public char[] dir_name = new char[11];
        public byte dir_attr;
        public byte[] dir_empty = new byte[12];
        public int dir_firstCluster;
        public int dir_filesize;
        public Directory_Entry()
        {

        }
        public Directory_Entry(string name, byte dir_attr, int dir_firstCluster)
        {
            this.dir_attr = dir_attr;
            if (dir_attr == 0x0)
            {
                string[] fileName = name.Split('.');
                assignFileName(fileName[0].ToCharArray(), fileName[1].ToCharArray());
            }
            else if (dir_attr == 0x10)
            {

                assignDIRName(name.ToCharArray());
            }
            this.dir_firstCluster = dir_firstCluster;
        }
        public void assignFileName(char[] name, char[] extension)
        {
            if (name.Length <= 7 && extension.Length == 3)
            {
                int j = 0;
                for (int i = 0; i < name.Length; i++)
                {
                    j++;
                    this.dir_name[i] = name[i];
                }
                j++;
                this.dir_name[j] = '.';
                for (int i = 0; i < extension.Length; i++)
                {
                    j++;
                    this.dir_name[j] = extension[i];
                }
                for (int i = ++j; i < dir_name.Length; i++)
                {
                    this.dir_name[i] = ' ';
                }
            }
            else
            {
                for (int i = 0; i < 7; i++)
                {
                    this.dir_name[i] = name[i];
                }
                this.dir_name[7] = '.';
                for (int i = 0, j = 8; i < extension.Length; j++, i++)
                {
                    this.dir_name[j] = extension[i];
                }
            }
        }
        public void assignDIRName(char[] name)
        {
            if (name.Length <= 11)
            {
                int j = 0;
                for (int i = 0; i < name.Length; i++)
                {
                    j++;
                    this.dir_name[i] = name[i];
                }
                for (int i = ++j; i < dir_name.Length; i++)
                {
                    this.dir_name[i] = ' ';
                }
            }
            else
            {
                int j = 0;
                for (int i = 0; i < 11; i++)
                {
                    j++;
                    this.dir_name[i] = name[i];
                }
            }
        }
    }
}
                                       +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Simple_Shell
{
    public class Directory : Directory_Entry
    {
        public List<Directory_Entry> DirOrFiles;
        public Directory parent;
        public Directory(string name, byte dir_attr, int dir_firstCluster, Directory pa) : base(name, dir_attr, dir_firstCluster)
        {
            DirOrFiles = new List<Directory_Entry>();
            Directory_Entry me = this.GetDirectory_Entry();
            DirOrFiles.Add(me);
            if (pa != null)
            {
                parent = pa;
                DirOrFiles.Add(this.parent.GetDirectory_Entry());
            }

        }
        public void updateContent(Directory_Entry d)
        {
            int index = searchDirectory(new string(d.dir_name));
            if (index != -1)
            {
                DirOrFiles.RemoveAt(index);
                DirOrFiles.Insert(index, d);
            }
        }
        public Directory_Entry GetDirectory_Entry()
        {
            Directory_Entry me = new Directory_Entry(new string(this.dir_name), this.dir_attr, this.dir_firstCluster);
            return me;
        }
        public void writeDirectory()
        {
            byte[] dirsorfilesBYTES = new byte[DirOrFiles.Count * 32];
            for (int i = 0; i < DirOrFiles.Count; i++)
            {
                byte[] b = Converter.Directory_EntryToBytes(this.DirOrFiles[i]);
                for (int j = i * 32, k = 0; k < b.Length; k++, j++)
                    dirsorfilesBYTES[j] = b[k];
            }
            List<byte[]> bytesls = Converter.splitBytes(dirsorfilesBYTES);
            int clusterFATIndex;
            if (this.dir_firstCluster != 0)
            {
                clusterFATIndex = this.dir_firstCluster;
            }
            else
            {
                clusterFATIndex = FAT.getAvilableCluster();
                this.dir_firstCluster = clusterFATIndex;
            }
            int lastCluster = -1;
            for (int i = 0; i < bytesls.Count; i++)
            {
                if (clusterFATIndex != -1)
                {
                    Virtual_Disk.writeCluster(bytesls[i], clusterFATIndex, 0, bytesls[i].Length);
                    FAT.setClusterPointer(clusterFATIndex, -1);
                    if (lastCluster != -1)
                        FAT.setClusterPointer(lastCluster, clusterFATIndex);
                    lastCluster = clusterFATIndex;
                    clusterFATIndex = FAT.getAvilableCluster();
                }
            }
            if (this.parent != null)
            {
                this.parent.updateContent(this.GetDirectory_Entry());
                this.parent.writeDirectory();
            }
            FAT.writeFAT();
        }
        public void readDirectory()
        {
            if (this.dir_firstCluster != 0)
            {
                DirOrFiles = new List<Directory_Entry>();
                int cluster = this.dir_firstCluster;
                int next = FAT.getClusterPointer(cluster);
                List<byte> ls = new List<byte>();
                do
                {
                    ls.AddRange(Virtual_Disk.readCluster(cluster));
                    cluster = next;
                    if (cluster != -1)
                        next = FAT.getClusterPointer(cluster);
                }
                while (next != -1);
                for (int i = 0; i < ls.Count; i++)
                {
                    byte[] b = new byte[32];
                    for (int k = i * 32, m = 0; m < b.Length && k < ls.Count; m++, k++)
                    {
                        b[m] = ls[k];
                    }
                    if (b[0] == 0)
                        break;
                    DirOrFiles.Add(Converter.BytesToDirectory_Entry(b));
                }
            }
        }
        public void deleteDirectory()
        {
            if (this.dir_firstCluster != 0)
            {
                int cluster = this.dir_firstCluster;
                int next = FAT.getClusterPointer(cluster);
                do
                {
                    FAT.setClusterPointer(cluster, 0);
                    cluster = next;
                    if (cluster != -1)
                        next = FAT.getClusterPointer(cluster);
                }
                while (cluster != -1);
            }
            if (this.parent != null)
            {
                int index = this.parent.searchDirectory(new string(this.dir_name));
                if (index != -1)
                {
                    this.parent.DirOrFiles.RemoveAt(index);
                    this.parent.writeDirectory();
                }
            }
            if (Program.current == this)
            {
                if (this.parent != null)
                {
                    Program.current = this.parent;
                    Program.currentPath = Program.currentPath.Substring(0, Program.currentPath.LastIndexOf('\\'));
                    Program.current.readDirectory();
                }
            }
            FAT.writeFAT();
        }
        public int searchDirectory(string name)
        {
            if (name.Length < 11)
            {
                name += "\0";
                for (int i = name.Length + 1; i < 12; i++)
                    name += " ";
            }
            else
            {
                name = name.Substring(0, 11);
            }
            for (int i = 0; i < DirOrFiles.Count; i++)
            {
                string n = new string(DirOrFiles[i].dir_name);
                if (n.Equals(name))
                    return i;
            }
            return -1;
        }
    }
}
    
    +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    
    
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Simple_Shell
{
    public class File_Entry : Directory_Entry
    {
        public string content;
        public Directory parent;
        public File_Entry(string name, byte dir_attr, int dir_firstCluster, Directory pa) : base(name, dir_attr, dir_firstCluster)
        {
            content = string.Empty;
            if (pa != null)
                parent = pa;
        }
        public Directory_Entry GetDirectory_Entry()
        {
            Directory_Entry me = new Directory_Entry(new string(this.dir_name), this.dir_attr, this.dir_firstCluster);
            return me;
        }
        public void writeFileContent()
        {
            byte[] contentBYTES = Converter.StringToBytes(content);
            List<byte[]> bytesls = Converter.splitBytes(contentBYTES);
            int clusterFATIndex;
            if (this.dir_firstCluster != 0)
            {
                clusterFATIndex = this.dir_firstCluster;
            }
            else
            {
                clusterFATIndex = FAT.getAvilableCluster();
                this.dir_firstCluster = clusterFATIndex;
            }
            int lastCluster = -1;
            for (int i = 0; i < bytesls.Count; i++)
            {
                if (clusterFATIndex != -1)
                {
                    Virtual_Disk.writeCluster(bytesls[i], clusterFATIndex, 0, bytesls[i].Length);
                    FAT.setClusterPointer(clusterFATIndex, -1);
                    if (lastCluster != -1)
                        FAT.setClusterPointer(lastCluster, clusterFATIndex);
                    lastCluster = clusterFATIndex;
                    clusterFATIndex = FAT.getAvilableCluster();
                }
            }
        }
        public void readFileContent()
        {
            if (this.dir_firstCluster != 0)
            {
                content = string.Empty;
                int cluster = this.dir_firstCluster;
                int next = FAT.getClusterPointer(cluster);
                List<byte> ls = new List<byte>();
                do
                {
                    ls.AddRange(Virtual_Disk.readCluster(cluster));
                    cluster = next;
                    if (cluster != -1)
                        next = FAT.getClusterPointer(cluster);
                }
                while (next != -1);
                content = Converter.BytesToString(ls.ToArray());
            }
        }
        public void deleteFile()
        {
            if (this.dir_firstCluster != 0)
            {
                int cluster = this.dir_firstCluster;
                int next = FAT.getClusterPointer(cluster);
                do
                {
                    FAT.setClusterPointer(cluster, 0);
                    cluster = next;
                    if (cluster != -1)
                        next = FAT.getClusterPointer(cluster);
                }
                while (cluster != -1);
            }
            if (this.parent != null)
            {
                int index = this.parent.searchDirectory(new string(this.dir_name));
                if (index != -1)
                {
                    this.parent.DirOrFiles.RemoveAt(index);
                    this.parent.writeDirectory();
                    FAT.writeFAT();
                }
            }
        }
    }
}
    
    +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
    using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace Simple_Shell
{
    public enum TokenType
    {
        Command, Not_Recognized, FullPathToDirectory, FileName, DirName, FullPathToFile
    }
    public struct Token
    {
        public TokenType key;
        public string value;
    }
    class Program
    {
        public static Directory current;
        public static string currentPath;
        static void Main(string[] args)
        {
            Console.WriteLine("Welcome to My Simple shell\n");
            Virtual_Disk.initalize("virtualDisk");
           // FAT.printFAT();
            currentPath = new string(current.dir_name);
            currentPath = currentPath.Trim(new char[] { '\0', ' ' });
            while (true)
            {
                Console.Write(currentPath + "\\" + ">");
                current.readDirectory();
                //for (int i = 0; i < current.DirOrFiles.Count; i++)
                //{
                //    Console.WriteLine(new string(current.DirOrFiles[i].dir_name) + $" fc={current.DirOrFiles[i].dir_firstCluster}");
                //}
                string input;
                input = Console.ReadLine();
                Parser.parse(input);
            }
        }
    }
}
