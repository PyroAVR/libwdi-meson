# Automate installation of USB device drivers in Windows
project(
    'libwdi',
    'c',
    version: '1.5.0',
)

windows = import('windows')

build_compiler = meson.get_compiler('c', native: true)
host_compiler = meson.get_compiler('c', native: false)
host_is_w64 = (host_compiler.get_define('__WIN64__') == '1' or host_compiler.get_define('_M_X64') != '')
host_is_arm64 = host_compiler.get_define('_M_ARM64') == '1'

libwdi_inc = [include_directories(['libwdi'])]

if host_compiler.get_id() == 'msvc'
    libwdi_inc += include_directories(['msvc'])
endif

## Create config.h, autotools header
conf_data = configuration_data()
if get_option('logging')
    conf_data.set('ENABLE_LOGGING')
endif

if get_option('debug_logging') == 'allowed'
    conf_data.set('INCLUDE_DEBUG_LOGGING')
elif get_option('debug_logging') == 'forced'
    conf_data.set('INCLUDE_DEBUG_LOGGING')
    conf_data.set('ENABLE_DEBUG_LOGGING')
endif

foreach header : ['dlfcn', 'inttypes', 'stdint', 'stdio', 'stdlib', 'strings', 'string', 'sys_stat', 'sys_types', 'unistd']
    conf_data.set10(header, host_compiler.has_header(header + '.h'))
endforeach

conf_data.set_quoted('package_name', 'libwdi')
conf_data.set_quoted('version', meson.project_version())
conf_data.set('wdf_ver', get_option('wdf_ver'))
# mesondefine will def/undef these correctly
if host_is_w64 and not host_is_arm64
    conf_data.set('OPT_M64', true)
elif host_is_arm64
    conf_data.set('OPT_ARM64', true)
else
    conf_data.set('OPT_M32', true)
endif


embed_driver_options = {
    'libusbk': get_option('libusbk_path'),
    'libusb0': get_option('libusb0_path'),
    'wdk': get_option('winusb_path'),
    'x64': get_option('wdk_x64_path'),
    # TODO what does this do on win10? options are "wdf" or "winusb".
    'coinstaller': 'wdf',
    'user': get_option('user_path'),
}

foreach def, opt : embed_driver_options
    if opt != ''
        conf_data.set_quoted((def + '_dir').to_upper(), opt)
    endif
endforeach


config_header = configure_file(
    input: 'config.h.in.meson',
    output: 'config.h',
    configuration: conf_data
)
config_header_dep = declare_dependency(sources: config_header)

## Installer

installer_name = 'installer_'
if host_is_w64
    installer_name += 'x64'
elif host_is_arm64
    installer_name += 'arm64'
else
    installer_name += 'x32'
endif

installer_sources = [files('libwdi' / 'installer.c')]
installer_deps = [config_header_dep]

if get_option('installer_use_elevation')
    # Embed a windows resource file into the libwdi-installer, to cause it to
    # run with elevated privileges by default.
    ins_winres_cfg = configuration_data()
    ins_winres_cfg.set('version', meson.project_version())
    ins_winres_cfg.set('installer_name', installer_name)
    ins_winres_cfg.set('elevation_manifest_path', meson.project_source_root() / 'elevation.manifest')
    ins_winres_file = configure_file(
        input: 'installer.rc.in.meson',
        output: 'installer.rc',
        configuration: ins_winres_cfg
    )
    installer_winres = windows.compile_resources(ins_winres_file)
    installer_winres_dep = declare_dependency(sources: installer_winres)

    installer_sources += installer_winres
    installer_deps += installer_winres_dep
endif

# Installer runs on the host machine, but is required to build the embedder,
# since it is part of the embedded contents.
installer = executable(installer_name,
    installer_sources,
    include_directories: libwdi_inc,
    link_args: [
        '-lsetupapi',
        '-lnewdev',
        '-lole32',
    ],
    dependencies: installer_deps,
)

## Embedder

# The embedder needs to run on the build machine: this program creates a header
# file that is the binary contents of the driver to install.
embedder_options = [
    '-DSRCDIR="' + meson.project_source_root() / 'libwdi"',
    '-DSOLUTIONDIR="' + meson.project_build_root() + '"',
]

# this is just to issue a build warning... meson version of the build will
# fail to execute from a 32-bit build machine.
if host_compiler.get_id() == 'msvc'
    if host_is_w64 or host_is_arm64
        embedder_options += '-DBUILD64'
    else
        embedder_options += '-DNO_BUILD64'
    endif
endif

embedder = executable('embedder',
    files(
        'libwdi' / 'embedder.c',
    ),
    include_directories: libwdi_inc,
    c_args: embedder_options,
    dependencies: [config_header_dep],
    native: true,
)

## libwdi

# Create the header file that will contain the drivers + driver installer
embedded_h = custom_target(
    'embedded.h',
    output: 'embedded.h',
    # the argument to embedder is just a name
    command: [embedder, '@OUTPUT@'],
    depends: [installer]
)
embedded_h_dep = declare_dependency(sources: embedded_h)

# windows resource file defines exported symbols of libwdi
libwdi_winres = windows.compile_resources(files('libwdi'/ 'libwdi.rc'))

libwdi = library('wdi',
    [
        files(
            'libwdi' / 'libwdi.c',
            'libwdi' / 'logging.c',
            'libwdi' / 'tokenizer.c',
            'libwdi' / 'libwdi_dlg.c',
            'libwdi' / 'pki.c',
            'libwdi' / 'vid_data.c',
        ),
        embedded_h,
        libwdi_winres
    ],
    link_args: [
        '-lsetupapi',
        '-lole32',
        '-lntdll',
        '-lnewdev',
    ],
    dependencies: [config_header_dep, embedded_h_dep],
    include_directories: libwdi_inc
)

libwdi_dep = declare_dependency(
    link_with: libwdi,
    include_directories: libwdi_inc
)

## Examples
if get_option('build_examples')
    examples_inc = include_directories('examples')

    ## Zadig

    # Graphical frontend for libwdi

    zadig_winres = windows.compile_resources(files('examples' / 'zadig.rc'))

    zadig = executable('zadig',
        [
            files(
                'examples' / 'zadig.c',
                'examples' / 'zadig_net.c',
                'examples' / 'zadig_parser.c',
                'examples' / 'zadig_stdlg.c',
                'examples' / 'profile.c',
            ),
            zadig_winres,
        ],
        link_args: [
            '-lsetupapi',
            '-lcomctl32',
            '-lwininet',
            '-lntdll',
            '-luuid',
            '-lole32',
        ],
        include_directories: [libwdi_inc, examples_inc],
        link_with: libwdi,
        gui_app: true,
    )

    ## libwdi-simple

    # console based frontend for libwdi

    libwdi_simple_winres = windows.compile_resources(files('examples' / 'wdi-simple.rc'))

    libwdi_simple = executable('libwdi_simple',
        [
            files(
                'examples' / 'wdi-simple.c',
                # getopt only used here, compiling it together w/ libwdi_simple
                'examples' / 'getopt' / 'getopt1.c',
                'examples' / 'getopt' / 'getopt.c'
            )
        ],
        include_directories: [libwdi_inc, examples_inc],
        link_with: libwdi,
        link_args: [
            '-lsetupapi'
        ],
        gui_app: false
    )
endif
