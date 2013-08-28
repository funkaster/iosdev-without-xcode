require 'yaml'
require 'rake/clean'

# Xcode 4.5+
SDK = "6.1"
DEV_TOOLS_HOME = "/Applications/Xcode.app/Contents/Developer"
ARCHS = {
	"device" => ["armv7"],
	"simulator" => ["i386"]
}
PLATFORMS = {
	"i386" => "iPhoneSimulator",
	"armv7" => "iPhoneOS",
	"armv7s" => "iPhoneOS"
}
SRC = FileList["src/*.c"] + FileList["src/*.cc"] + FileList["src/*.m"]
OBJ = SRC.map { |s|
	s.pathmap("%{src,build}X.o")
}
CLEAN.include("build", "out")

module Builder
	def self.cc; @cc end
	def self.lipo; @lipo end
	def self.codesign_allocate; @codesign_allocate end
	def self.dsymutil; @dsymutil end
	def self.cflags; @cflags.join(" ") end
	def self.cxxflags; @cxxflags.join(" ") end
	def self.ldflags; @ldflags.join(" ") end
	def self.plistbuddy; "/usr/libexec/PlistBuddy" end
	def self.base_flags; @base_flags.join(" ") end
	def self.current_arch; @arch end

	def self.sysroot_for_platform(platform)
		[
			"#{DEV_TOOLS_HOME}/Platforms/#{platform}.platform/Developer/SDKs/#{platform}#{SDK}.sdk",
			"#{DEV_TOOLS_HOME}/Platforms/#{platform}.platform/Developer/SDKs/#{platform}#{SDK}.sdk/usr/include",
			"#{DEV_TOOLS_HOME}/Platforms/#{platform}.platform/Developer/SDKs/#{platform}#{SDK}.sdk/System/Library/Frameworks",
			(platform == "iPhoneOS" ? "iphoneos" : "ios-simulator")
		]
	end

	def self.default_warnings
		%w(
			-Wno-trigraphs
			-fpascal-strings
			-Wno-missing-field-initializers
			-Wno-missing-prototypes
			-Wreturn-type
			-Wno-non-virtual-dtor
			-Wno-overloaded-virtual
			-Wno-exit-time-destructors
			-Wformat
			-Wno-missing-braces
			-Wparentheses
			-Wswitch
			-Wno-unused-function
			-Wno-unused-label
			-Wno-unused-parameter
			-Wunused-variable
			-Wunused-value
			-Wempty-body
			-Wuninitialized
			-Wno-unknown-pragmas
			-Wno-shadow
			-Wno-four-char-constants
			-Wno-conversion
			-Wno-shorten-64-to-32
			-Wno-c++11-extensions
			-fmessage-length=0
		)
	end

	def self.set_current_arch(arch)
		@arch = arch
	end

	def self.load_conf
		@conf = YAML.load(File.read("project.yaml"))
	end

	def self.prepare_for_current_arch(debug=false)
		# test for /usr/libexec/PlistBuddy
		raise "Missing /usr/libexec/PlistBuddy" unless File.exists?("/usr/libexec/PlistBuddy")
		@cc = "#{DEV_TOOLS_HOME}/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang"
		@lipo = "#{DEV_TOOLS_HOME}/Platforms/iPhoneOS.platform/Developer/usr/bin/lipo"
		@codesign_allocate = "#{DEV_TOOLS_HOME}/Platforms/iPhoneOS.platform/Developer/usr/bin/codesign_allocate"
		@dsymutil = "#{DEV_TOOLS_HOME}/Toolchains/XcodeDefault.xctoolchain/usr/bin/dsymutil"
		sysroot, base_include, sdk_root, short_name = self.sysroot_for_platform(PLATFORMS[@arch])
		@base_flags = [
			"-arch", @arch,
			"-m#{short_name}-version-min=#{@conf['minios']}",
			"-DTARGET_OS_IPHONE",
			"-DUSE_FILE32API",
			"-isysroot", sysroot,
			"-F", base_include,
			"-F", sdk_root
		]
		@cflags = self.default_warnings + @base_flags
		@cflags += ["-g"] if ENV['DEBUG']
		@cflags += ["-fobjc-abi-version=2"] if @conf["link_objc_runtime"]
		@ldflags = [
			"-Xlinker",
			"-no_implicit_dylibs"
		]
		@ldflags += ["-fobjc-link-runtime", "-objc_abi_version"] if @conf["link_objc_runtime"]
		@ldflags += ["-fobjc-arc"] if @conf["objc_arc"]
		@conf['frameworks'].each do |f|
			@ldflags += ["-framework #{f}"]
		end
		@cxxflags = [
			"-std=#{@conf['cxxstd']}",
			"-stdlib=#{@conf['stdlib']}"
		]
	end

	# convenient access to @conf properties
	def self.method_missing(m_id, *args, &block)
		if @conf.key?(m_id.to_s)
			return @conf[m_id.to_s]
		end
		super
	end
end

Builder.load_conf

def find_source(objfile)
	file = objfile.pathmap("%{build/#{Builder.current_arch},src}X")
	SRC.find { |s| s.ext("") == file }
end

def lang_for_compiler(source)
	ext = source.match(/\.(\w+)$/)
	case ext[1]
	when "cpp"
		return "c++"
	when "mm"
		return "objective-c++"
	when "c"
		return "c"
	when "m"
		return "objective-c"
	end
	raise "invalid source type: #{source}"
end

rule ".o" => lambda{ |objfile| find_source(objfile) } do |t|
	cc = Builder.cc
	flags = [Builder.cflags]
	if t.source.match(/\.(cpp|mm)$/)
		flags += [Builder.cxxflags]
	end
	flags += ["-x", lang_for_compiler(t.source)]
	sh "#{cc} #{flags.join(" ")} -c #{t.source} -o #{t.name}"
end

directory "build"
directory "out"

# links a binary
def link_current_binary(objs)
	# here we need to link the object files into the final binary
	cc = Builder.cc
	ldflags = Builder.ldflags
	name = Builder.project_name
	sh "env IPHONEOS_DEPLOYMENT_TARGET=#{Builder.minios} #{cc} #{Builder.base_flags} #{ldflags} #{objs.join(" ")} -o build/#{Builder.current_arch}/#{name}"
end

# signs the bundle/binary with the right identity
def sign_bundle(bundle)
	name = Builder.project_name
	cp "#{DEV_TOOLS_HOME}/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS#{SDK}.sdk/ResourceRules.plist",
		"out/#{name}.app"
	args = [
		"--force",
		"--sign", "\"#{Builder.sign_identity}\"",
		"--entitlements", "myapp.xcent",
		"--resource-rules=out/#{name}.app/ResourceRules.plist",
		"out/#{name}.app"
	]
	new_path = File.dirname(Builder.codesign_allocate) + ":" + ENV["PATH"]
	sh "env CODESIGN_ALLOCATE=#{Builder.codesign_allocate} PATH=#{new_path} /usr/bin/codesign #{args.join(" ")}"
end

# creates the bundle - called as a dependency task. Do not call directly
task :bundle do |t|
	name = Builder.project_name
	# if in DEBUG, generate app.dSYM
	if ENV['DEBUG']
		sh "#{Builder.dsymutil} out/#{name}.app/#{name} -o out/#{name}.app.dSYM"
	end
	# copy resources
	cp_r FileList["resources/*"], "out/#{name}.app/"
	# process Info.plist
	sh "#{Builder.plistbuddy} -c \"Set :CFBundleDisplayName #{Builder.bundle_name}\" out/#{name}.app/Info.plist"
	sh "#{Builder.plistbuddy} -c \"Set :CFBundleExecutable #{name}\" out/#{name}.app/Info.plist"
	sh "#{Builder.plistbuddy} -c \"Set :CFBundleIdentifier #{Builder.bundle_identifier}\" out/#{name}.app/Info.plist"
	sh "#{Builder.plistbuddy} -c \"Set :CFBundleName #{Builder.project_name}\" out/#{name}.app/Info.plist"
	sh "#{Builder.plistbuddy} -c \"Add :MinimumOSVersion string #{Builder.minios}\" out/#{name}.app/Info.plist"
	# convert to binary
	sh "plutil -convert binary1 out/#{name}.app/Info.plist"
end

desc "create the app bundle (platforms: #{ARCHS.keys}) Pass the platform as env PLATFORM"
task :app => ["build", "out"] do |t|
	platform = ENV['PLATFORM'] || "simulator"
	archs = ARCHS[platform]
	archs.each do |arch|
		Builder.set_current_arch(arch)
		Builder.prepare_for_current_arch
		mkdir_p "build/#{arch}"
		objs = OBJ.map { |o| o.pathmap("%{build,build/#{Builder.current_arch}}X.o") }
		task "binary_#{arch}".to_sym => objs do |t|
			link_current_binary(objs)
		end
		Rake::Task["binary_#{arch}".to_sym].invoke
	end
	name = Builder.project_name
	mkdir_p "out/#{name}.app"
	# link the binary, if there's more than one arch, lipo them together
	if archs.size > 1
		binaries = archs.map { |arch| "build/#{arch}/#{name}" }
		sh "#{Builder.lipo} -create #{binaries.join(' ')} -output out/#{name}.app/#{name}"
	else
		cp "build/#{Builder.current_arch}/#{name}", "out/#{name}.app/"
	end
	# finish the bundle
	Rake::Task[:bundle].invoke
	# if the platform is "device", then sign the binary
	sign_bundle("out/#{name}.app") if platform == "device"
end

desc "run the app in the simulator (by default it launches an iphone retina)"
task :simulate, [:is_retina, :device_family] => [:app] do |t, args|
	args.with_defaults(:is_retina => true, :device_family => "iphone")
	name = Builder.project_name
	sh "./ios-sim launch out/#{name}.app #{args.is_retina ? "--retina" : ""} --family #{args.device_family}"
end

desc "run the app in the device"
task :run => [:app] do |t|
	name = Builder.project_name
	sh "./fruitstrap -d -b out/#{name}.app"
end

desc "creates a SublimeText project"
task :sublime do |t|
	project_path = "#{Builder.project_name}.sublime-project"
	if not File.exists?(project_path)
		str = <<-EOS
{
	"folders":
	[
		{
			"path": "#{ENV['PWD']}",
			"folder_exclude_patterns": ["out", "build"],
			"file_exclude_patterns": ["*.sublime-*"]
		}
	],
	"settings":
	{
		"sublimeclang_options":
		[
			"-DTARGET_OS_IPHONE",
			"-DUSE_FILE32API",
			"-mios-simulator-version-min=#{Builder.minios}",
			"-isysroot", "#{DEV_TOOLS_HOME}/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator#{SDK}.sdk",
			"-F", "#{DEV_TOOLS_HOME}/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator#{SDK}.sdk/usr/include",
			"-F", "#{DEV_TOOLS_HOME}/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator#{SDK}.sdk/System/Library/Frameworks"
		]
	}
}
		EOS
		File.open(project_path, "w+") { |f| f.write str }
	end
end

desc "bootstrap your directory (create an empty iOS app in the current dir)"
task :bootstrap do
end
